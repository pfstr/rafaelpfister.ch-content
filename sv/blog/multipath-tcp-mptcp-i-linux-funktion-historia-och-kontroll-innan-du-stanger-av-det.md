---
title: "Multipath TCP (MPTCP) i Linux: funktion, historia och kontroll innan du stänger av det"
navTitle: "MPTCP i Linux"
description: "Multipath TCP fördelar en TCP-anslutning över flera nätverkssökvägar. Sedan Go 1.24 lyssnar många Go-tjänster i Linux via MPTCP utan att någon har konfigurerat det. Med CVE-2026-98432 har detta blivit relevant. Artikeln förklarar protokollet, dess historia och hur du kontrollerar om du kan stänga av MPTCP."
date: "2026-11-02"
kategorie: "Linux-kärnan"
timeToRead: "10 min läsning"
themen:
  - linux-kernel
  - sicherheitsluecken
produkte:
  - "linux-kernel"
  - "sicherheitsluecken"
protokolle:
  - "tcp"
  - "haertung"
hauptthema: "linux-kernel"
slug: "multipath-tcp-mptcp-i-linux-funktion-historia-och-kontroll-innan-du-stanger-av-det"
translationId: "article-1caab37cb0d7fce7"
translationOf: multipath-tcp-mptcp-erklaert
url: https://rafaelpfister.ch/sv/blog/multipath-tcp-mptcp-i-linux-funktion-historia-och-kontroll-innan-du-stanger-av-det
translationSourceHash: 0c0b43321d6db2e6da54028fa5339a41dcd664ca8d2264fec44309b32d7809b1
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:33:38.263Z
translationReview: automatic
---

När jag kontrollerade min hemserver för CVE-2026-98432 upptäckte jag att flera tjänster lyssnade via Multipath TCP, bland annat Prometheus-node-exporter, Pi-hole-exporter och `tailscaled`, alla skrivna i Go. Jag hade aldrig konfigurerat detta. Den hittills offentligt beskrivna root-exploiten för denna sårbarhet förutsätter just en sådan MPTCP-lyssnare.

Kort sammanfattning: MPTCP ingår i Linux officiella kärna sedan kärna 5.6 och är tillåtet som standard. Sedan Go 1.24 öppnar Go-program automatiskt sina lyssnare som MPTCP-sockets. Om du kan stänga av MPTCP kan avgöras med två kommandon och några räknare i kärnan. På de flesta servrar använder ingen klient protokollet, och efter avstängning faller Go automatiskt tillbaka till vanlig TCP.

## Vad Multipath TCP är

En vanlig TCP-anslutning är bunden till exakt ett adresspar: käll-IP och port, mål-IP och port. När en smartphone byter från Wi-Fi till mobilnät ändras käll-IP-adressen och anslutningen bryts. Multipath TCP utökar TCP så att en anslutning kan använda flera sökvägar samtidigt eller efter varandra, exempelvis Wi-Fi och mobilnät.

Tekniskt består en MPTCP-anslutning av flera `Subflows`. Varje subflow är för nätverket en vanlig TCP-anslutning över en sökväg. Applikationen märker inget av detta: den arbetar med en enda socket, och kärnan fördelar data över subflows. För att data ska komma fram i rätt ordning har MPTCP ett eget 64-bitars sekvensnummer på anslutningsnivå (Data Sequence Number), utöver sekvensnumren för de enskilda subflows.

MPTCP förhandlas fram i den vanliga TCP-handshake-processen. SYN, SYN/ACK och det första ACK innehåller TCP-alternativet `MP_CAPABLE` (Option Kind 30), där båda sidor anger att de talar MPTCP och utbyter nycklar. Ytterligare subflows ansluter senare till anslutningen med alternativet `MP_JOIN`. Om `MP_CAPABLE` saknas i svaret, eftersom motparten inte känner till MPTCP eller en middlebox tar bort alternativet, fortsätter anslutningen som vanlig TCP. Detta fallback ingår i standarden och är anledningen till att MPTCP-kompatibla servrar fungerar med alla TCP-klienter.

I Linux är MPTCP opt-in per socket: en applikation måste uttryckligen skapa socketen med protokollet `IPPROTO_MPTCP` (värde 262). Ett program som inte gör det fortsätter att använda vanlig TCP, även om MPTCP är aktivt i kärnan.

## Historien

MPTCP uppstod vid universitet och togs i drift i stor skala via Apple. Linux-implementeringen låg utanför den officiella kärnan i över ett decennium innan den togs in i ny form.

| Tidpunkt | Händelse |
|---|---|
| från cirka 2009 | Forskare vid Université catholique de Louvain (UCLouvain) utvecklar den första Linux-implementeringen som en egen kärngren (multipath-tcp.org). |
| mars 2011 | RFC 6182 beskriver arkitekturen för Multipath TCP (Informational). |
| januari 2013 | RFC 6824 specificerar MPTCP version 0 som ett experimentellt protokoll. |
| september 2013 | iOS 7 släpps med MPTCP, den första kommersiella användningen i stor skala. Apple använder bland annat MPTCP för Siri. |
| juni 2015 | KT (Korea Telecom) startar en kommersiell tjänst som sammanför Wi-Fi och LTE med MPTCP och möjliggör upp till 1 Gbit/s för smartphones; presenterad vid IETF 93. |
| 2017 | iOS 11 öppnar MPTCP för apputvecklare via `URLSession` (`multipathServiceType`). |
| mars 2020 | RFC 8684 definierar MPTCP version 1 som Proposed Standard och ersätter RFC 6824. |
| 29 mars 2020 | Linux 5.6 släpps med ett första MPTCP-stöd i den officiella kärnan, omskrivet och endast för version 1. Ytterligare subflows tillkom i följande versioner. |
| 2020 | 3GPP specificerar ATSSS (Access Traffic Steering, Switching, Splitting) för 5G i Release 16; där fördelar MPTCP trafik över mobilnät och Wi-Fi. |
| augusti 2023 | Go 1.21 introducerar `SetMultipathTCP` för dialers och lyssnare, avstängt som standard. |
| februari 2025 | Go 1.24 aktiverar MPTCP som standard för lyssnare (`ListenConfig`), medan dialers fortsätter använda TCP. |
| oktober 2026 | CVE-2026-98432: Use-after-free i IPv4-TCP-koden; den visade root-exploiten kräver en MPTCP-lyssnare. |

UCLouvains out-of-tree-gren stödde version 0 och 1. Den officiella kärnan känner endast till version 1; projektet rekommenderar numera enbart denna implementering.

## Varför MPTCP är aktivt på många servrar

Två standardinställningar sammanfaller. Kärnan tillåter MPTCP-sockets som standard: sysctl-inställningen `net.mptcp.enabled` är satt till `1`. Och Go-program från version 1.24 skapar varje lyssnare som en MPTCP-socket, om kärnan tillåter det. Go-tjänster som Prometheus-exporter eller Tailscale lyssnar därför via MPTCP så snart de har byggts med Go 1.24 eller senare.

För vanliga klienter förändras inget: de skickar inget `MP_CAPABLE`, och anslutningen körs som TCP. Den extra attackytan uppstår dock på serversidan, eftersom varje klient som skickar med `MP_CAPABLE` når MPTCP-koden i kärnan.

## CVE-2026-98432

CVE-2026-98432 är en use-after-free vid upprättande av IPv4-TCP-anslutningar med IP-alternativ (CVSS 3.1: 9.8). Enligt CVE-posten påverkas Linux från version 4.4. Felet i sig påverkar varje TCP-lyssnare, inte bara MPTCP. För exploatering till root-rättigheter har dock hittills endast en kedja via en MPTCP-lyssnare visats; enligt lilting.ch demonstrerade Anthropic den i en labbmiljö den 27 oktober 2026. Om root-rättigheter även kan nås utan MPTCP är oklart.

Uppströms är felet åtgärdat i de stabila kärnorna 5.10.272, 5.15.223, 6.1.190, 6.6.159, 6.12.113, 6.18.56 och 7.2.10 samt i mainline 7.3 (commit ”ipv4: free inet_opt and ireq_opt after an RCU grace period"). Ubuntu angav den 30 oktober 2026 fortfarande LTS-versionerna 22.04, 24.04 och 26.04 som ”Vulnerable, work in progress”; för 26.04 är fixen planerad i `linux` 7.0.0-43.43. Distributioner bakportar fixar; därför är paketversionen i distributionens advisory avgörande, inte kärnans huvudversion.

Att stänga av MPTCP tar bort förutsättningen för den visade exploiten. Sårbarheten i IPv4-TCP-koden kvarstår tills den uppdaterade kärnan har installerats och startats.

## Kontrollera om MPTCP är aktivt och används

### 1. Tillåter kärnan MPTCP?

```bash
sysctl net.mptcp.enabled
```

`1` betyder att applikationer får skapa MPTCP-sockets. Om `sysctl` meddelar att nyckeln är okänd är MPTCP inte kompilerat i kärnan och inte tillgängligt på denna värd.

### 2. Vilka tjänster lyssnar via MPTCP?

```bash
sudo ss -Mlnp
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-M` | visa endast MPTCP-sockets |
| `-l` | visa endast lyssnande sockets (lyssnare) |
| `-n` | adresser och portar numeriskt, utan namnuppslagning |
| `-p` | visa processen för socketen; kräver root-rättigheter för andra användares processer |

</details>

Varje rad är en MPTCP-lyssnare. Samma port visas dessutom med `ss -tln` som TCP-lyssnare; det är normalt eftersom kärnan internt har en TCP-socket för varje MPTCP-lyssnare. Processnamnen visar vilka program som påverkas. Vanligtvis är det Go-program.

### 3. Har någon klient någonsin använt MPTCP?

Kärnan räknar MPTCP-handshakes sedan senaste starten. De relevanta räknarna:

```bash
nstat -asz |
  grep -E 'MPTcpExt(MPCapable|MPCurrEstab)'
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `-a` | absoluta värden sedan systemstart i stället för skillnad sedan senaste anrop |
| `-s` | uppdatera inte historikfilen från `nstat` |
| `-z` | skriv även ut räknare med värdet 0 |

</details>

| Räknare | Betydelse |
|---|---|
| `MPTcpExtMPCapableSYNRX` | inkommande SYN med `MP_CAPABLE` till en MPTCP-lyssnare |
| `MPTcpExtMPCapableACKRX` | inkommande MPTCP-anslutningar vars handshake slutfördes |
| `MPTcpExtMPCapableSYNTX` | utgående SYN med `MP_CAPABLE`, alltså anslutningar som denna värd ville upprätta via MPTCP |
| `MPTcpExtMPCapableSYNACKRX` | utgående anslutningar där motparten bekräftade MPTCP |
| `MPTcpExtMPCapableFallbackACK` | anslutningar som på serversidan föll tillbaka till TCP under handshaken |
| `MPTcpExtMPCapableFallbackSYNACK` | anslutningar som på klientsidan föll tillbaka |
| `MPTcpExtMPCurrEstab` | MPTCP-anslutningar som för närvarande finns |

Om `MPCapableSYNRX` och `MPCapableACKRX` är 0 efter en representativ drifttid har ingen klient använt MPTCP. Om `MPCapableSYNTX` är större än 0 upprättar en applikation på värden själv utgående MPTCP-anslutningar; identifiera den applikationen innan avstängning. Befintliga MPTCP-anslutningar visas av `ss -Mani` med detaljer om subflows.

Räknarna gäller sedan senaste omstarten. En server som startades för bara en timme sedan ger därför inget tillförlitligt resultat. Dessutom räknar kärnan per nätverksnamnområde, mer om detta i nästa avsnitt.

### 4. Kontrollera containrar separat

`net.mptcp.enabled` gäller per nätverksnamnområde. Varje container med eget nätverk (Docker-bridge, Podman, Kubernetes-pods) har ett eget värde, och ett nytt namnområde börjar med standardvärdet `1`, oberoende av inställningen på värden. På min hemserver stod alla bridge-containrar fortfarande på `1` efter avstängningen på värden; endast containrarna med `network_mode: host` övertog `0`.

```bash
for c in $(docker ps -q); do
  pid=$(docker inspect -f '{{.State.Pid}}' "$c")
  name=$(docker inspect -f '{{.Name}}' "$c")
  echo "$name $(sudo nsenter -t "$pid" -n sysctl -n net.mptcp.enabled)"
done
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `docker ps -q` | ID:n för alla körande containrar |
| `docker inspect -f '{{.State.Pid}}'` | PID för huvudprocessen i containern, via vilken `nsenter` hittar namnområdet |
| `nsenter -t <pid>` | gå in i den angivna processens namnområden |
| `-n` | gå endast in i nätverksnamnområdet; filsystem och verktyg förblir värdens |
| `sysctl -n` | skriv endast ut värdet, utan nyckelnamn |

</details>

`nsenter` har fördelen att varken `sysctl` eller `cat` behöver finnas i containern; många images (distroless, scratch) innehåller inget skal. På samma sätt kan `ss -Mln` och räknarna från `nstat` frågas i en containers namnområde.

## När du kan stänga av MPTCP

Avstängning är oproblematisk om räknarna från steg 3 på värden och i containrarna inte visar någon MPTCP-användning. Lyssnande Go-program förlorar inget på detta: Go försöker skapa MPTCP-socketen, får ett fel från kärnan och skapar tyst en TCP-socket. Andra program som uttryckligen begär `IPPROTO_MPTCP` får felet `ENOPROTOOPT`. Om de sedan faller tillbaka till TCP eller avbryter beror på programvaran; testa sådana tjänster efter ändringen.

Försiktighet krävs i följande fall:

- **Hybrid Access och länkbuntning:** Routrar eller proxyservrar som sammanför DSL, fiber och mobilnät via MPTCP (exempelvis OpenMPTCProuter eller Hybrid Access-gateways från leverantörer) är beroende av MPTCP.
- **Servrar för mobilappar:** En egen iOS-app med `multipathServiceType` drar nytta av nätverksbyten endast om servern talar MPTCP. Räknaren `MPCapableACKRX` visar om sådana klienter förekommer.
- **Utgående MPTCP:** Program som startas med `mptcpize` eller som körs i Go med `GODEBUG=multipathtcp=1` eller `3` upprättar utgående MPTCP. Detta syns genom att `MPCapableSYNTX` är större än 0.

## Stänga av MPTCP

Det finns tre möjligheter, beroende på önskad omfattning.

### För hela värden

```bash
echo "net.mptcp.enabled = 0" |
  sudo tee /etc/sysctl.d/99-no-mptcp.conf
sudo sysctl --system
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `/etc/sysctl.d/99-no-mptcp.conf` | konfigurationsfil som läses vid uppstart; inställningen kvarstår efter omstarter |
| `sysctl --system` | läs in alla filer under `/etc/sysctl.d/` och övriga standardsökvägar på nytt och tillämpa dem omedelbart |

</details>

Inställningen förhindrar endast att nya MPTCP-sockets skapas. Redan öppna lyssnare förblir MPTCP-sockets tills tjänsten startas om. En omstart av värden är enklast; därefter måste `ss -Mln` ge en tom lista.

### För containrar

I containrar med eget nätverk anger du värdet per container. Docker tillåter sysctls från området `net.*`, så länge containern inte använder värdnätverket:

```yaml
services:
  app:
    sysctls:
      net.mptcp.enabled: 0
```

Vid direkt anrop motsvarar det `docker run --sysctl net.mptcp.enabled=0 …`. Ändringen träder i kraft nästa gång containern skapas om (`docker compose up -d`).

### Endast för enskilda Go-program

I stället för att stänga av MPTCP i kärnan kan Go-standardbeteendet återställas per tjänst. Miljövariabeln `GODEBUG=multipathtcp=0` stänger av MPTCP för lyssnare och dialers. För en systemd-tjänst görs detta via en drop-in:

```bash
sudo systemctl edit node_exporter
```

```ini
[Service]
Environment=GODEBUG=multipathtcp=0
```

Om tjänsten redan sätter en `GODEBUG`-variabel kombineras posterna med kommatecken i en variabel. Denna möjlighet fungerar endast för Go-program och endast om koden inte själv anropar `SetMultipathTCP`; ett uttryckligt anrop i programmet har företräde framför `GODEBUG`. För egna Go-projekt finns samma inställning som `godebug multipathtcp=0` i `go.mod` eller som `//go:debug multipathtcp=0` i huvudpaketet.

## Kontroll efter ändringen

```bash
sysctl net.mptcp.enabled
ss -Mln
```

Det förväntade resultatet är `0` och ingen rad under tabellhuvudet i `ss`. Tjänsterna lyssnar därefter fortfarande på samma portar, synliga med `ss -tln`. Ändringen kan återställas genom att ta bort filen under `/etc/sysctl.d/` och starta om värden.

## Källor

1.  [RFC 8684: TCP Extensions for Multipath Operation with Multiple Addresses](https://datatracker.ietf.org/doc/html/rfc8684): Specifikation av MPTCP version 1 (mars 2020) med subflows, `MP_CAPABLE`, Data Sequence Number och fallback.

2.  [RFC 6824](https://datatracker.ietf.org/doc/rfc6824/): MPTCP version 0, experimentell (januari 2013), ersatt av RFC 8684.

3.  [RFC 6182: Architectural Guidelines for Multipath TCP Development](https://datatracker.ietf.org/doc/rfc6182/): Arkitekturdokument från mars 2011.

4.  [Linux Kernel Documentation: Multipath TCP](https://docs.kernel.org/networking/mptcp.html): Opt-in via `IPPROTO_MPTCP`, felkoder vid skapande av sockets, Path Manager.

5.  [Linux Kernel Documentation: MPTCP Sysfs variables](https://docs.kernel.org/networking/mptcp-sysctl.html): Betydelse och standardvärde för `net.mptcp.enabled` och övriga MPTCP-sysctls.

6.  [mptcp.dev](https://www.mptcp.dev/): Upstream-projektets webbplats med FAQ, felsökningsguide (`ss -Mani`, `nstat`) och information om macOS och iOS.

7.  [KernelNewbies: Linux 5.6](https://kernelnewbies.org/Linux_5.6): Releaseöversikt med det första MPTCP-stödet i den officiella kärnan.

8.  [Go 1.21 Release Notes](https://go.dev/doc/go1.21): Introduktion av MPTCP-stöd i paketet `net`, avstängt som standard, med tyst fallback till TCP.

9.  [Go 1.24 Release Notes](https://go.dev/doc/go1.24): MPTCP som standard för `ListenConfig` under Linux.

10.  [Go: GODEBUG-inställningar](https://go.dev/doc/godebug): Värdena 0 till 3 för `multipathtcp` samt `godebug` i `go.mod` och `//go:debug`.

11.  [Wikipedia: Multipath TCP](https://en.wikipedia.org/wiki/Multipath_TCP): Översikt över implementeringar och användningar (UCLouvain, Apple, KT, 3GPP ATSSS).

12.  [Ubuntu Security: CVE-2026-98432](https://ubuntu.com/security/CVE-2026-98432): Status för Ubuntu-kärnor per release och planerade fixversioner.

13.  [lilting.ch: CVE-2026-98432 Linux TCP Use-After-Free](https://lilting.ch/en/articles/linux-ipv4-tcp-options-uaf-mptcp-rce): Bedömning av exploaterbarheten, MPTCP:s roll och fixversioner för stabila kärnor.
