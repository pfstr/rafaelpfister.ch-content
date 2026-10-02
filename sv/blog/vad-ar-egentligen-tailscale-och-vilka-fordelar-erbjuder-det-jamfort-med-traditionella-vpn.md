---
title: "Vad är egentligen Tailscale och vilka fördelar erbjuder det jämfört med traditionella VPN-anslutningar?"
navTitle: "Tailscale vs. VPN"
description: "Tailscale bygger ett mesh-VPN ovanpå WireGuard, där enheter ansluter direkt till varandra i stället för via en central VPN-koncentrator. Så samverkar Coordination Server, NAT-traversering och DERP-reläer, vilka fördelar finns jämfört med IPsec- och SSL-VPN, och vilka beroenden och begränsningar bör du känna till före användning."
date: "2026-10-01"
kategorie: "VPN och fjärråtkomst"
timeToRead: "11 min läsning"
themen:
  - vpn-fernzugriff
produkte:
  - "tailscale"
protokolle:
  - "tcp"
  - "haertung"
slug: "vad-ar-egentligen-tailscale-och-vilka-fordelar-erbjuder-det-jamfort-med-traditionella-vpn"
translationId: "article-91fddf1e0239f5c4"
aiPrompt: |
  Du bist mein Netzwerk-Assistent. Hilf mir einzuschätzen, ob Tailscale unser bestehendes VPN ganz oder teilweise ersetzen kann: Ist-Zustand aufnehmen (VPN-Gateway, Benutzer, Standorte, erreichbare Netze), Zugriffsregeln nach dem Prinzip der minimalen Rechte als Tailscale-Policy entwerfen, Subnet Router und Exit Nodes planen und Abhängigkeiten wie Identity Provider, Datenschutz nach revDSG und Koexistenz mit anderen VPN-Clients prüfen.
translationOf: tailscale-vorteile-vpn
url: https://rafaelpfister.ch/sv/blog/vad-ar-egentligen-tailscale-och-vilka-fordelar-erbjuder-det-jamfort-med-traditionella-vpn
translationSourceHash: 965ff1000bee1a9e9899d6ffd750d28abbea09dd50896551eae369b7bf20f162
translationModel: gpt-5.6-terra
translatedAt: 2026-10-02T10:06:45.228Z
translationReview: required
---

# Vad är egentligen Tailscale och vilka fördelar erbjuder det jämfört med traditionella VPN-anslutningar?

Tailscale är en VPN-tjänst som ansluter enheter till ett privat nätverk, ett så kallat Tailnet. Tekniskt bygger den på WireGuard. Skillnaden mot ett klassiskt företags-VPN ligger i topologin: Enheterna upprättar sina krypterade tunnlar direkt med varandra (mesh). En central VPN-koncentrator som all trafik passerar genom behövs inte. Endast administrationen är centraliserad: En Coordination Server distribuerar offentliga nycklar, adresser och åtkomstregler, men ser själv inga nyttodata.

För att komma igång räcker det med ett konto hos en Identity Provider (Microsoft, Google, GitHub, Apple eller en OIDC-leverantör) och klienten på varje enhet. Portvidarebefordran i brandväggen behövs normalt inte. Därför är Tailscale vanligt både för hemnätverk och för fjärråtkomst till servrar och kundsystem.

## Så fungerar ett traditionellt VPN

Ett klassiskt VPN för fjärråtkomst fungerar enligt hub-and-spoke-principen. Vid företagsnätverkets gräns finns en VPN-gateway (brandvägg eller appliance) som är nåbar från internet. Klienten på den bärbara datorn upprättar en tunnel till denna gateway; vanliga alternativ är IPsec/IKEv2 (UDP 500 och 4500), SSL-VPN-varianter via TCP 443 eller OpenVPN. Efter inloggning får enheten en adress från en pool och rutter till de interna nätverken.

Denna modell har fungerat väl i årtionden, men medför strukturella egenskaper som orsakar arbete i dagens drift:

- **Offentligt nåbar gateway:** VPN-koncentratorn måste vara nåbar från internet och är därmed ett prioriterat angreppsmål. Sårbarheter i VPN-appliances har upprepade gånger utnyttjats aktivt under de senaste åren; den amerikanska myndigheten CISA beordrade till och med i januari 2024, genom Emergency Directive 24-01, att berörda Ivanti-gateways skulle kopplas bort från nätet.
- **Single point of failure och flaskhals:** All trafik passerar gatewayen. Om den slutar fungera eller bandbredden är uttömd påverkas alla användare samtidigt.
- **Omvägar (hairpinning):** Om två medarbetare arbetar hemifrån mot samma server i molnet går trafiken först till datacentret och sedan ut därifrån igen.
- **Grova åtkomsträttigheter:** Efter att anslutningen har upprättats får enheten ofta åtkomst till ett helt nätverkssegment. Finkorniga regler per användare och tjänst är möjliga, men underhålls sällan konsekvent.
- **Arbete med site-to-site:** Varje ytterligare plats behöver en egen tunnel med samordnade parametrar (Phase-1/Phase-2-proposals, Pre-Shared Keys eller certifikat, fasta offentliga IP-adresser).

## Så är Tailscale uppbyggt

Tailscale separerar Control Plane från Data Plane. Control Plane är Coordination Server, som Tailscale driver som molntjänst. Data Plane består av WireGuard-tunnlarna mellan enheterna.

| Komponent | Uppgift |
|---|---|
| Klient (`tailscaled`) | Skapar nyckelparet lokalt, upprättar WireGuard-tunnlar till de andra noderna, tillämpar åtkomstreglerna lokalt |
| Coordination Server | Autentiserar enheter via Identity Provider, distribuerar offentliga nycklar, adresser, DNS-inställningar och policyn till alla noder |
| DERP-reläer | Vidarebefordrar krypterade paket när ingen direkt anslutning kan upprättas |
| Peer Relays | Egna enheter i Tailnet som fungerar som relä med högre genomströmning; de prioriteras före DERP |

En enhets privata nyckel lämnar aldrig enheten. Coordination Server känner endast till de offentliga nycklarna och kan därför inte dekryptera trafiken. Varje enhet får en fast adress från området `100.64.0.0/10` (adressutrymmet för Carrier-Grade NAT) samt en IPv6-adress från `fd7a:115c:a1e0::/48`. Via MagicDNS är enheterna dessutom nåbara under sitt namn, till exempel `nas` eller `nas.tailnet-name.ts.net`.

### NAT-traversering: varför inga portvidarebefordringar behövs

De flesta enheter finns bakom en NAT-router eller brandvägg och är inte direkt nåbara utifrån. Tailscale löser detta med NAT-traversering: Båda sidor fastställer via STUN sin offentliga adress och tilldelade port, utbyter denna information via Coordination Server och skickar samtidigt UDP-paket till varandra. De utgående paketen öppnar en tillståndspost i båda brandväggarna, genom vilken motpartens paket sedan kan komma in (UDP Hole Punching).

Om det inte lyckas, till exempel vid restriktiva brandväggar som blockerar utgående UDP eller vid vissa former av Carrier-Grade NAT, går trafiken via ett DERP-relä över HTTPS. Även då förblir den end-to-end-krypterad med WireGuard; reläet ser endast krypterade paket. Priset är högre latens och lägre genomströmning. Peer Relays minskar denna nackdel genom att en egen enhet med god anslutning tar över vidarebefordringen.

## Fördelarna jämfört med ett traditionellt VPN

| Kriterium | Traditionellt VPN | Tailscale |
|---|---|---|
| Topologi | Hub-and-spoke via en central gateway | Mesh, direkta anslutningar mellan enheter |
| Inkommande portar | Gatewayen måste vara nåbar från internet | Inga inkommande portvidarebefordringar behövs |
| Autentisering | Lokala konton, RADIUS, certifikat, ofta separat MFA | Inloggning via befintlig Identity Provider inklusive dess MFA |
| Åtkomsträttigheter | Ofta per nätverkssegment | Per användare, grupp, enhet och port i en central policy |
| Ny plats | Site-to-site-tunnel med samordnade parametrar | Installera klient eller Subnet Router |
| Avbrott i centrum | Ingen åtkomst längre för alla | Befintliga anslutningar fortsätter fungera; nya enheter och policyändringar väntar |
| Protokoll | IPsec, SSL-VPN, OpenVPN | WireGuard |

### Mindre attackyta

Eftersom klienterna upprättar sina anslutningar utgående behöver ingen enhet ha en öppen port på internet. En server som endast ska vara nåbar via Tailnet kan binda sina tjänster exklusivt till Tailscale-gränssnittet. Då är den osynlig för portskannrar på internet. WireGuard självt är med omkring 4 000 rader kärnkod betydligt mindre än typiska IPsec- eller SSL-VPN-implementationer och använder en fast uppsättning moderna metoder (Curve25519, ChaCha20-Poly1305, BLAKE2s). Det finns ingen förhandling av Cipher Suites, vilket vid IPsec regelbundet leder till felkonfigurationer.

### Identitet i stället för nätverksadress

Varje enhet i Tailnet är knuten till en användare eller en tagg. Inloggningen sker via den Identity Provider som du redan använder; MFA och Conditional Access från Microsoft Entra ID gäller därmed även för nätverksåtkomst. Nya enheter kan endast ansluta till Tailnet efter lyckad inloggning hos Identity Provider, och varje enhetsnyckel upphör som standard att gälla efter 180 dagar. Om en medarbetare lämnar företaget spärrar du kontot hos Identity Provider; med SCIM-provisionering (från Standard-planen) inaktiveras användaren automatiskt i Tailnet och dess enheter förlorar åtkomst.

### Åtkomstregler enligt principen om minsta behörighet

Som standard kan varje enhet i ett nytt Tailnet nå varje annan enhet. För produktionsbruk definierar du åtkomsträttigheterna i en central policyfil (HuJSON). Följande exempel tillåter administratörsgruppen SSH och HTTPS till alla servrar med taggen `tag:server`, medan alla övriga användare endast får HTTPS till intranätservern:

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
<summary>Förklaring av alternativ</summary>

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

Eftersom tunnlarna finns direkt mellan enheterna tar trafiken den kortaste vägen. Två enheter på samma kontor kommunicerar lokalt, och en bärbar dator på hemmakontoret når en molnserver direkt. Det minskar latensen och avlastar huvudplatsens internetanslutning.

### Ansluta befintliga nätverk

Alla enheter kan inte köra en Tailscale-klient, exempelvis skrivare, äldre NAS-system eller industristyrningar. I sådana fall tar en Subnet Router över rollen som gateway: En Linux-server i målnätet annonserar det lokala subnätet i Tailnet (advertise), och behöriga enheter når det via denna.

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
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `curl -fsSL …/install.sh \| sh` | Hämtar det officiella installationsskriptet och konfigurerar distributionens paketförråd |
| `net.ipv4.ip_forward = 1` | Tillåter Linux-kärnan att vidarebefordra paket mellan gränssnitt; utan denna inställning fungerar ingen Subnet Router |
| `sysctl -p <datei>` | Läser in inställningen omedelbart utan omstart |
| `tailscale up` | Anmäler enheten till Tailnet; vid första anropet visas en inloggningslänk |
| `--advertise-routes=<subnetz>` | Annonserar det angivna subnätet i Tailnet; flera subnät avgränsas med kommatecken |
| `--advertise-tags=<tag>` | Tilldelar enheten en tagg som åtkomstreglerna hänvisar till |

</details>

Den annonserade rutten måste därefter godkännas i administratörskonsolen, om ingen automatisk godkänning (`autoApprovers`) har konfigurerats i policyn. På motsvarande sätt kan en enhet drivas som Exit Node med `--advertise-exit-node`. Klienter som väljer denna Exit Node vidarebefordrar då all sin internettrafik via den, vilket motsvarar Full Tunnel-läget i ett klassiskt VPN.

### Mindre driftsarbete

Nyckelrotation, adressallokering, DNS och routing hanteras av Coordination Server. En ny plats behöver en Subnet Router med internetanslutning, men ingen fast offentlig IP-adress och ingen samordning av IPsec-parametrar med motparten. För fjärråtkomst till servrar finns dessutom Tailscale SSH (inloggning via Tailnet-identitet utan distribuerade SSH-nycklar) och `tailscale serve` (delning av en lokal webbtjänst i Tailnet).

## Begränsningar och beroenden

Tailscale ersätter den klassiska VPN-gatewayen, men flyttar en del av ansvaret till en extern tjänst. Du bör kontrollera dessa punkter före en införande.

| Ämne | Att beakta |
|---|---|
| Beroende av leverantören | Control Plane är en molntjänst från Tailscale Inc. Vid avbrott fortsätter befintliga anslutningar fungera, men nya enheter, inloggningar och policyändringar är inte möjliga förrän tjänsten har återställts |
| Metadata | Enhetsnamn, Tailnet-adresser, offentliga IP-adresser, användarkonton och anslutningstidpunkter behandlas hos leverantören; nyttodata behandlas inte |
| Förtroende för nyckeldistributionen | Coordination Server bestämmer vilka offentliga nycklar en enhet accepterar. Tailnet Lock kräver dessutom en signatur från betrodda egna enheter |
| Källkod | Klientkärnan är öppen källkod (BSD-3-Clause), Coordination Server är det inte. Headscale är ett alternativ med öppen källkod för egen drift med begränsad funktionalitet |
| Adresskonflikter | `100.64.0.0/10` används även av vissa leverantörer för Carrier-Grade NAT och av andra VPN-produkter; överlappningar leder till routingproblem |
| Samexistens med andra VPN-klienter | En andra VPN-klient med Full Tunnel kan omdirigera Tailscale-trafiken. Området `100.64.0.0/10` och Tailscale-tjänsten måste då undantas från tunneln |
| Prestanda via reläer | Om ingen direkt anslutning kan upprättas sjunker genomströmningen märkbart via DERP; `tailscale netcheck` visar om UDP utåt fungerar |
| Ingen ersättning för webbfiltrering | Tailscale reglerar åtkomst till interna resurser. Det hanterar inte innehållsfiltrering och inspektion av internettrafik |

För företag i Schweiz gäller den reviderade dataskyddslagen (revDSG). Eftersom användarnas personuppgifter (konton, enheter, anslutningsmetadata) uppstår hos leverantören ska du ta upp Tailscale i registret över behandlingsaktiviteter, om ditt företag måste föra ett sådant register (från 250 medarbetare eller vid behandlingar med hög risk). Kontrollera leverantörens personuppgiftsbiträdesavtal (Data Processing Addendum) och den rättsliga grunden för utlämnande till utlandet enligt art. 16 revDSG, exempelvis en certifiering enligt Swiss-U.S. Data Privacy Framework eller standardavtalsklausuler. Om databehandling hos leverantören är utesluten återstår Headscale som egen driven Control Plane.

> **EU-anmärkning:** För filialer i EU gäller GDPR och dess regler för överföring till tredjeland (art. 44 ff. GDPR). Bedömningen är innehållsmässigt liknande; där är EU-U.S. Data Privacy Framework avgörande.

## Kostnader

Planen Personal är kostnadsfri och omfattar upp till sex användare, valfritt antal användarenheter och 50 taggade enheter (status oktober 2026). För företagsanvändning kostar planen Standard 8 USD och Premium 18 USD per användare och månad. Premium omfattar bland annat Network Flow Logs, loggstreaming och utökade alternativ för Tailscale SSH. Enterprise erbjuds individuellt.

## Vem Tailscale passar för

Tailscale passar särskilt när användare och resurser är distribuerade: hemmakontor, molnservrar hos flera leverantörer, små externa platser utan fast IP-adress eller fjärrunderhåll hos kunder. För små och medelstora företag kan det helt ersätta en VPN-gateway. I större miljöer körs det ofta parallellt med det befintliga VPN:et, till exempel för administrativ åtkomst till servrar, där de finkorniga åtkomstreglerna ger störst nytta.

Det är mindre lämpligt när krav föreskriver en helt egen driven infrastruktur och Headscale inte täcker funktionaliteten, eller när all internettrafik måste filtreras centralt. I så fall behövs fortfarande en Secure Web Gateway som kompletterar Tailscale.

För att testa räcker det att installera klienten på två enheter och logga in med samma konto. Med `tailscale status` ser du sedan alla enheter i Tailnet, och med `tailscale ping <gerät>` kontrollerar du om en direkt anslutning eller ett relä används.

## Källor

1.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): arkitektur med Coordination Server, WireGuard-mesh och lokal tillämpning av reglerna.

2.  [Tailscale: How NAT traversal works](https://tailscale.com/blog/how-nat-traversal-works): utförlig förklaring av STUN, UDP Hole Punching och de fall där ett relä behövs.

3.  [Tailscale Docs: DERP servers](https://tailscale.com/kb/1232/derp-servers): funktion och kryptering för reläservrarna.

4.  [Tailscale Docs: Tailscale Peer Relays](https://tailscale.com/kb/1591/peer-relays): egna enheter som relä med prioritet före DERP.

5.  [Tailscale Docs: Grants](https://tailscale.com/kb/1324/grants): syntax för åtkomstregler i policyfilen.

6.  [Tailscale Docs: Subnet routers](https://tailscale.com/kb/1019/subnets): konfigurering av Subnet Routers inklusive IP-forwarding och godkännande av rutterna.

7.  [Tailscale Docs: Exit nodes](https://tailscale.com/kb/1103/exit-nodes): dirigera all internettrafik via en enhet i Tailnet.

8.  [Tailscale Docs: Tailnet Lock](https://tailscale.com/kb/1226/tailnet-lock): signering av nya enheter genom egna, betrodda noder.

9.  [Tailscale: Pricing](https://tailscale.com/pricing): planer och gränser, hämtat den 1 oktober 2026.

10.  [WireGuard: Next Generation Kernel Network Tunnel (Whitepaper)](https://www.wireguard.com/papers/wireguard.pdf): protokolldesign och använda kryptografiska metoder.

11.  [GitHub: tailscale/tailscale](https://github.com/tailscale/tailscale): klientens källkod under BSD-3-Clause.

12.  [GitHub: juanfont/headscale](https://github.com/juanfont/headscale): implementering med öppen källkod av Coordination Server för egen drift.

13.  [CISA: Emergency Directive 24-01](https://www.cisa.gov/news-events/directives/ed-24-01-mitigate-ivanti-connect-secure-and-ivanti-policy-secure-vulnerabilities): order om frånkoppling av sårbara Ivanti-VPN-gateways i januari 2024.

14.  [Fedlex: Federal Act on Data Protection (FADP)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): art. 16 om utlämnande av personuppgifter till utlandet.
