---
title: "Multipath TCP (MPTCP) på Linux: virkemåte, historie og kontroll før deaktivering"
navTitle: "MPTCP på Linux"
description: "Multipath TCP fordeler en TCP-forbindelse over flere nettverksbaner. Siden Go 1.24 lytter mange Go-tjenester på Linux via MPTCP uten at noen har satt det opp. Med CVE-2026-98432 har dette blitt relevant. Artikkelen forklarer protokollen, historien og hvordan du kontrollerer om du kan deaktivere MPTCP."
date: "2026-11-02"
kategorie: "Linux-kjerne"
timeToRead: "10 min lesetid"
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
slug: "multipath-tcp-mptcp-pa-linux-virkemate-historie-og-kontroll-for-deaktivering"
translationId: "article-1caab37cb0d7fce7"
translationOf: multipath-tcp-mptcp-erklaert
url: https://rafaelpfister.ch/no/blog/multipath-tcp-mptcp-pa-linux-virkemate-historie-og-kontroll-for-deaktivering
translationSourceHash: 0c0b43321d6db2e6da54028fa5339a41dcd664ca8d2264fec44309b32d7809b1
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:34:17.957Z
translationReview: automatic
---

Da jeg kontrollerte hjemmeserveren min for CVE-2026-98432, oppdaget jeg at flere tjenester lyttet via Multipath TCP, deriblant Prometheus-node-exporter, Pi-hole-exporter og `tailscaled`, alle skrevet i Go. Jeg hadde aldri satt dette opp. Den hittil offentlig beskrevne root-utnyttelsen for denne sårbarheten forutsetter nettopp en slik MPTCP-lytter.

Kort oppsummert: MPTCP har vært i den offisielle Linux-kjernen siden kjerne 5.6 og er tillatt som standard. Siden Go 1.24 åpner Go-programmer automatisk lytterne sine som MPTCP-sokler. Om du kan deaktivere MPTCP, kan avklares med to kommandoer og noen tellere i kjernen. På de fleste servere bruker ingen klient protokollen, og Go faller automatisk tilbake til vanlig TCP etter deaktivering.

## Hva Multipath TCP er

En vanlig TCP-forbindelse er bundet til nøyaktig ett adressepar: kilde-IP og port, mål-IP og port. Bytter en smarttelefon fra WLAN til mobilnett, endres kilde-IP-en, og forbindelsen brytes. Multipath TCP utvider TCP slik at én forbindelse kan bruke flere baner samtidig eller etter hverandre, for eksempel WLAN og mobilnett.

Teknisk består en MPTCP-forbindelse av flere `Subflows`. Hver subflow er en vanlig TCP-forbindelse over én bane for nettet. Applikasjonen merker ingenting til dette: Den arbeider med én enkelt sokkel, og kjernen fordeler dataene på subflowene. For at dataene skal komme frem i riktig rekkefølge, bruker MPTCP et eget 64-bits sekvensnummer på forbindelsesnivå (Data Sequence Number), i tillegg til sekvensnumrene til de enkelte subflowene.

MPTCP forhandles i den vanlige TCP-handshaken. SYN, SYN/ACK og den første ACK-en inneholder TCP-opsjonen `MP_CAPABLE` (Option Kind 30), der begge sider erklærer at de snakker MPTCP og utveksler nøkler. Flere subflower slutter seg senere til forbindelsen med opsjonen `MP_JOIN`. Mangler `MP_CAPABLE` i svaret, fordi motparten ikke kjenner MPTCP eller en middlebox fjerner opsjonen, fortsetter forbindelsen som vanlig TCP. Denne tilbakefallsfunksjonen er en del av standarden og grunnen til at MPTCP-kompatible servere fungerer med alle TCP-klienter.

På Linux er MPTCP opt-in per sokkel: En applikasjon må uttrykkelig opprette sokkelen med protokollen `IPPROTO_MPTCP` (verdi 262). Et program som ikke gjør dette, bruker fortsatt vanlig TCP, selv om MPTCP er aktivt i kjernen.

## Historien

MPTCP oppstod ved universiteter og ble tatt i bruk i stor skala via Apple. Linux-implementeringen var utenfor den offisielle kjernen i over et tiår før den ble tatt inn i ny form.

| Tidspunkt | Hendelse |
|---|---|
| fra rundt 2009 | Forskere ved Université catholique de Louvain (UCLouvain) utvikler den første Linux-implementeringen som en egen kjernegren (multipath-tcp.org). |
| mars 2011 | RFC 6182 beskriver arkitekturen til Multipath TCP (Informational). |
| januar 2013 | RFC 6824 spesifiserer MPTCP versjon 0 som en eksperimentell protokoll. |
| september 2013 | iOS 7 lanseres med MPTCP, den første kommersielle bruken i stort omfang. Apple bruker blant annet MPTCP for Siri. |
| juni 2015 | KT (Korea Telecom) starter en kommersiell tjeneste som samler WLAN og LTE via MPTCP og muliggjør opptil 1 Gbit/s på smarttelefoner; presentert på IETF 93. |
| 2017 | iOS 11 åpner MPTCP for apputviklere via `URLSession` (`multipathServiceType`). |
| mars 2020 | RFC 8684 definerer MPTCP versjon 1 som Proposed Standard og erstatter RFC 6824. |
| 29. mars 2020 | Linux 5.6 lanseres med en første MPTCP-støtte i den offisielle kjernen, skrevet på nytt og bare for versjon 1. Ytterligere subflower kom i de følgende versjonene. |
| 2020 | 3GPP spesifiserer ATSSS (Access Traffic Steering, Switching, Splitting) for 5G i Release 16; MPTCP fordeler der trafikk over mobilnett og WLAN. |
| august 2023 | Go 1.21 innfører `SetMultipathTCP` for dialere og lyttere, deaktivert som standard. |
| februar 2025 | Go 1.24 aktiverer MPTCP for lyttere som standard (`ListenConfig`), dialere forblir på TCP. |
| oktober 2026 | CVE-2026-98432: Use-after-free i IPv4-TCP-koden; den viste root-utnyttelsen trenger en MPTCP-lytter. |

UCLouvains out-of-tree-gren støttet versjon 0 og 1. Den offisielle kjernen kjenner bare versjon 1; prosjektet anbefaler i dag utelukkende denne implementeringen.

## Hvorfor MPTCP er aktivt på mange servere

To standardinnstillinger sammenfaller. Kjernen tillater MPTCP-sokler som standard: Sysctl-innstillingen `net.mptcp.enabled` er satt til `1`. Og Go-programmer fra versjon 1.24 oppretter hver lytter som en MPTCP-sokkel, dersom kjernen tillater det. Go-tjenester som Prometheus-eksportører eller Tailscale lytter derfor via MPTCP så snart de er bygget med Go 1.24 eller nyere.

For vanlige klienter endrer dette ingenting: De sender ikke `MP_CAPABLE`, og forbindelsen går som TCP. Den ekstra angrepsflaten oppstår imidlertid på serversiden, fordi enhver klient som sender med `MP_CAPABLE`, når MPTCP-koden i kjernen.

## CVE-2026-98432

CVE-2026-98432 er en use-after-free ved opprettelse av IPv4-TCP-forbindelser med IP-opsjoner (CVSS 3.1: 9.8). Ifølge CVE-oppføringen er Linux fra versjon 4.4 berørt. Selve feilen påvirker alle TCP-lyttere, ikke bare MPTCP. For utnyttelse til root-rettigheter er det imidlertid hittil bare vist en kjede via en MPTCP-lytter; ifølge lilting.ch demonstrerte Anthropic den 27. oktober 2026 i et laboratoriemiljø. Om root-rettigheter også kan oppnås uten MPTCP, er uavklart.

Oppstrøms er feilen rettet i de stabile kjernene 5.10.272, 5.15.223, 6.1.190, 6.6.159, 6.12.113, 6.18.56 og 7.2.10 samt i mainline 7.3 (commit «ipv4: free inet_opt and ireq_opt after an RCU grace period"). Ubuntu førte 30. oktober 2026 fortsatt LTS-versjonene 22.04, 24.04 og 26.04 som «Vulnerable, work in progress"; for 26.04 er rettelsen planlagt i `linux` 7.0.0-43.43. Distribusjoner tilbakeporterer rettelser; derfor er pakkeversjonen i distribusjonens sikkerhetsråd avgjørende, ikke kjernens hovedversjon.

Å deaktivere MPTCP fjerner forutsetningen for den viste utnyttelsen. Sårbarheten i IPv4-TCP-koden består til den oppdaterte kjernen er installert og startet.

## Kontroller om MPTCP er aktivt og brukes

### 1. Tillater kjernen MPTCP?

```bash
sysctl net.mptcp.enabled
```

`1` betyr at applikasjoner kan opprette MPTCP-sokler. Dersom `sysctl` melder at nøkkelen er ukjent, er MPTCP ikke kompilert inn i kjernen og ikke tilgjengelig på denne verten.

### 2. Hvilke tjenester lytter via MPTCP?

```bash
sudo ss -Mlnp
```

<details class="options-details">
<summary>Forklaring av opsjoner</summary>

| Opsjon | Virkning |
|---|---|
| `-M` | vis bare MPTCP-sokler |
| `-l` | vis bare lyttende sokler (lyttere) |
| `-n` | adresser og porter numerisk, uten navneoppløsning |
| `-p` | vis prosessen til sokkelen; krever root-rettigheter for prosesser til andre brukere |

</details>

Hver linje er en MPTCP-lytter. Den samme porten vises i tillegg med `ss -tln` som en TCP-lytter; dette er normalt, fordi kjernen internt fører en TCP-sokkel for hver MPTCP-lytter. Prosessnavnene viser hvilke programmer som er berørt. Vanligvis er dette Go-programmer.

### 3. Har noen klient noen gang brukt MPTCP?

Kjernen teller MPTCP-handshaker siden siste oppstart. De relevante tellerne:

```bash
nstat -asz |
  grep -E 'MPTcpExt(MPCapable|MPCurrEstab)'
```

<details class="options-details">
<summary>Forklaring av opsjoner</summary>

| Opsjon | Virkning |
|---|---|
| `-a` | absolutte verdier siden systemoppstart i stedet for differansen siden forrige kall |
| `-s` | ikke oppdater historikkfilen til `nstat` |
| `-z` | skriv også ut tellere med verdien 0 |

</details>

| Teller | Betydning |
|---|---|
| `MPTcpExtMPCapableSYNRX` | innkommende SYN med `MP_CAPABLE` til en MPTCP-lytter |
| `MPTcpExtMPCapableACKRX` | innkommende MPTCP-forbindelser med fullført handshake |
| `MPTcpExtMPCapableSYNTX` | utgående SYN med `MP_CAPABLE`, altså forbindelser denne verten ønsket å opprette via MPTCP |
| `MPTcpExtMPCapableSYNACKRX` | utgående forbindelser der motparten har bekreftet MPTCP |
| `MPTcpExtMPCapableFallbackACK` | forbindelser som på serversiden falt tilbake til TCP under handshaken |
| `MPTcpExtMPCapableFallbackSYNACK` | forbindelser som på klientsiden falt tilbake |
| `MPTcpExtMPCurrEstab` | MPTCP-forbindelser som finnes nå |

Hvis `MPCapableSYNRX` og `MPCapableACKRX` står på 0 etter en representativ driftstid, har ingen klient brukt MPTCP. Er `MPCapableSYNTX` større enn 0, oppretter en applikasjon på verten selv utgående MPTCP-forbindelser; denne applikasjonen bør du identifisere før deaktivering. Eksisterende MPTCP-forbindelser vises av `ss -Mani` med detaljer om subflowene.

Tellerne gjelder siden siste omstart. En server som først ble startet for en time siden, gir derfor ingen pålitelig vurdering. Kjernen teller dessuten per nettverksnavnerom, mer om dette i neste avsnitt.

### 4. Kontroller containere separat

`net.mptcp.enabled` gjelder per nettverksnavnerom. Hver container med eget nettverk (Docker-bro, Podman, Kubernetes-poder) har sin egen verdi, og et nytt navnerom starter med standardverdien `1`, uavhengig av innstillingen på verten. På hjemmeserveren min stod alle bridge-containere fortsatt på `1` etter deaktivering på verten; bare containere med `network_mode: host` overtok `0`.

```bash
for c in $(docker ps -q); do
  pid=$(docker inspect -f '{{.State.Pid}}' "$c")
  name=$(docker inspect -f '{{.Name}}' "$c")
  echo "$name $(sudo nsenter -t "$pid" -n sysctl -n net.mptcp.enabled)"
done
```

<details class="options-details">
<summary>Forklaring av opsjoner</summary>

| Opsjon | Virkning |
|---|---|
| `docker ps -q` | ID-er til alle kjørende containere |
| `docker inspect -f '{{.State.Pid}}'` | PID-en til hovedprosessen i containeren, som `nsenter` bruker til å finne navnerommet |
| `nsenter -t <pid>` | gå inn i navnerommene til den angitte prosessen |
| `-n` | gå bare inn i nettverksnavnerommet; filsystem og verktøy forblir vertens |
| `sysctl -n` | skriv bare ut verdien, uten nøkkelnavn |

</details>

`nsenter` har den fordelen at verken `sysctl` eller `cat` trenger å finnes i containeren; mange images (distroless, scratch) inneholder ingen shell. På samme måte kan `ss -Mln` og tellerne fra `nstat` spørres i navnerommet til en container.

## Når du kan deaktivere MPTCP

Deaktivering er uproblematisk dersom tellerne fra trinn 3 på verten og i containerne ikke viser MPTCP-bruk. Lyttende Go-programmer mister ingenting av den grunn: Go forsøker MPTCP-sokkelen, mottar en feil fra kjernen og oppretter stille en TCP-sokkel. Andre programmer som uttrykkelig ber om `IPPROTO_MPTCP`, mottar feilen `ENOPROTOOPT`. Om de da faller tilbake til TCP eller avbryter, avhenger av programvaren; test slike tjenester etter endringen.

Vær forsiktig i disse tilfellene:

- **Hybrid Access og lenkebunting:** Rutere eller proxyer som samler DSL, fiber og mobilnett via MPTCP (for eksempel OpenMPTCProuter eller Hybrid Access-gatewayer fra tilbydere), er avhengige av MPTCP.
- **Servere for mobilapper:** En egen iOS-app med `multipathServiceType` har bare nytte ved nettverksbytte dersom serveren snakker MPTCP. Tellerne `MPCapableACKRX` viser om slike klienter forekommer.
- **Utgående MPTCP:** Programmer som startes med `mptcpize` eller kjører i Go med `GODEBUG=multipathtcp=1` eller `3`, oppretter utgående MPTCP. Dette kan gjenkjennes ved at `MPCapableSYNTX` er større enn 0.

## Deaktivere MPTCP

Det finnes tre muligheter, avhengig av ønsket omfang.

### For hele verten

```bash
echo "net.mptcp.enabled = 0" |
  sudo tee /etc/sysctl.d/99-no-mptcp.conf
sudo sysctl --system
```

<details class="options-details">
<summary>Forklaring av opsjoner</summary>

| Opsjon | Virkning |
|---|---|
| `/etc/sysctl.d/99-no-mptcp.conf` | konfigurasjonsfil som leses ved oppstart; innstillingen beholdes etter omstarter |
| `sysctl --system` | last inn alle filer under `/etc/sysctl.d/` og de øvrige standardbanene på nytt, og bruk dem umiddelbart |

</details>

Innstillingen hindrer bare opprettelsen av nye MPTCP-sokler. Allerede åpne lyttere forblir MPTCP-sokler til tjenesten startes på nytt. En omstart av verten er enklest; deretter må `ss -Mln` vise en tom liste.

### For containere

I containere med eget nettverk setter du verdien per container. Docker tillater sysctl-er fra området `net.*`, så lenge containeren ikke bruker vertsnettverket:

```yaml
services:
  app:
    sysctls:
      net.mptcp.enabled: 0
```

Ved direkte kall tilsvarer dette `docker run --sysctl net.mptcp.enabled=0 …`. Endringen trer i kraft neste gang containeren opprettes på nytt (`docker compose up -d`).

### Bare for enkelte Go-programmer

I stedet for å deaktivere MPTCP i kjernen kan Go-standardatferden tilbakestilles per tjeneste. Miljøvariabelen `GODEBUG=multipathtcp=0` deaktiverer MPTCP for lyttere og dialere. For en systemd-tjeneste gjøres dette med et drop-in:

```bash
sudo systemctl edit node_exporter
```

```ini
[Service]
Environment=GODEBUG=multipathtcp=0
```

Hvis tjenesten allerede setter en `GODEBUG`-variabel, kombineres oppføringene med komma i én variabel. Denne muligheten virker bare for Go-programmer og bare dersom koden ikke selv kaller `SetMultipathTCP`; et eksplisitt kall i programmet har forrang over `GODEBUG`. For egne Go-prosjekter er den samme innstillingen tilgjengelig som `godebug multipathtcp=0` i `go.mod` eller som `//go:debug multipathtcp=0` i hovedpakken.

## Kontroll etter endringen

```bash
sysctl net.mptcp.enabled
ss -Mln
```

Det forventes `0` og ingen linje under tabelloverskriften til `ss`. Tjenestene lytter fortsatt på de samme portene, synlig med `ss -tln`. Endringen kan reverseres ved å slette filen under `/etc/sysctl.d/` og starte verten på nytt.

## Kilder

1.  [RFC 8684: TCP Extensions for Multipath Operation with Multiple Addresses](https://datatracker.ietf.org/doc/html/rfc8684): Spesifikasjon av MPTCP versjon 1 (mars 2020) med subflower, `MP_CAPABLE`, Data Sequence Number og tilbakefall.

2.  [RFC 6824](https://datatracker.ietf.org/doc/rfc6824/): MPTCP versjon 0, eksperimentell (januar 2013), erstattet av RFC 8684.

3.  [RFC 6182: Architectural Guidelines for Multipath TCP Development](https://datatracker.ietf.org/doc/rfc6182/): Arkitekturdokument fra mars 2011.

4.  [Linux Kernel Documentation: Multipath TCP](https://docs.kernel.org/networking/mptcp.html): Opt-in via `IPPROTO_MPTCP`, feilkoder ved opprettelse av sokler, Path Manager.

5.  [Linux Kernel Documentation: MPTCP Sysfs variables](https://docs.kernel.org/networking/mptcp-sysctl.html): Betydning og standardverdi for `net.mptcp.enabled` og de øvrige MPTCP-sysctl-ene.

6.  [mptcp.dev](https://www.mptcp.dev/): Nettstedet til oppstrømsprosjektet med FAQ, feilsøkingsveiledning (`ss -Mani`, `nstat`) og informasjon om macOS og iOS.

7.  [KernelNewbies: Linux 5.6](https://kernelnewbies.org/Linux_5.6): Utgivelsesoversikt med den første MPTCP-støtten i den offisielle kjernen.

8.  [Go 1.21 Release Notes](https://go.dev/doc/go1.21): Innføring av MPTCP-støtte i pakken `net`, deaktivert som standard, med stille tilbakefall til TCP.

9.  [Go 1.24 Release Notes](https://go.dev/doc/go1.24): MPTCP som standard for `ListenConfig` på Linux.

10.  [Go: GODEBUG-innstillinger](https://go.dev/doc/godebug): Verdiene 0 til 3 for `multipathtcp` samt `godebug` i `go.mod` og `//go:debug`.

11.  [Wikipedia: Multipath TCP](https://en.wikipedia.org/wiki/Multipath_TCP): Oversikt over implementeringer og brukstilfeller (UCLouvain, Apple, KT, 3GPP ATSSS).

12.  [Ubuntu Security: CVE-2026-98432](https://ubuntu.com/security/CVE-2026-98432): Status for Ubuntu-kjernene per utgivelse og planlagte rettelsesversjoner.

13.  [lilting.ch: CVE-2026-98432 Linux TCP Use-After-Free](https://lilting.ch/en/articles/linux-ipv4-tcp-options-uaf-mptcp-rce): Vurdering av utnyttbarheten, rollen til MPTCP og rettelsesversjoner av de stabile kjernene.
