---
title: "Tailscale: Sammenligning av Exit-Node og subnettruter, og hvordan de fungerer teknisk"
navTitle: "Exit-Node vs. subnett"
description: "Exit-Node og subnettruter er to beslektede, men ulike driftsmoduser i Tailscale. En subnettruter åpner målrettet bestemte IP-områder, mens en Exit-Node leder all internettrafikk gjennom seg. Hva forskjellen betyr i praksis, hvordan Tailscale implementerer dette med WireGuard, rutegodkjenning og SNAT, og hvor grensene går for hver variant."
date: "2026-09-02"
kategorie: "Nettverk og VPN"
timeToRead: "11 min. lesetid"
themen:
  - tailscale
produkte:
  - "tailscale"
protokolle:
  - "tcp"
  - "haertung"
slug: "tailscale-sammenligning-av-exit-node-og-subnettruter-og-hvordan-de-fungerer-teknisk"
translationId: "article-c26cca4d635b9a04"
translationOf: tailscale-exit-node-subnet-routes
translationSourceHash: cefcefd45f617dfb668890986aaad256dbeca2b84d7c3472cd9eb5d6bb049911
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:54:14.868Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/no/blog/tailscale-sammenligning-av-exit-node-og-subnettruter-og-hvordan-de-fungerer-teknisk
---

# Tailscale: Sammenligning av Exit-Node og subnettruter, og hvordan de fungerer teknisk

En Tailscale-node er i utgangspunktet bare seg selv: tilgjengelig via Tailscale-adressen sin, og ikke noe mer. For at en node skal gi andre enheter tilgang til mer enn bare seg selv, finnes det to driftsmoduser som ofte forveksles: **subnettruteren** og **Exit-Node**. Begge utvider rekkevidden til en node, men i ulike retninger. Den som kjenner forskjellen, velger riktig variant og unngår å sende all trafikk gjennom en fremmed datamaskin ved en feil.

Kortversjonen: En subnettruter åpner **målrettet bestemte IP-områder** bak noden, for eksempel det lokale nettverket med en NAS og en skriver. En Exit-Node leder **all internettrafikk** fra en enhet gjennom seg, slik som en klassisk full-tunnel-VPN. Begge bygger teknisk på samme mekanisme: annonsering av ruter. Exit-Node er i bunn og grunn et spesialtilfelle av subnettruteren, der standardruten annonseres.

## Subnettruter: målrettet tilgang til et nettverk

En subnettruter annonserer ett eller flere IP-områder som den kan nå i det lokale nettverket. Andre enheter i tailnettet som godtar disse rutene, kan nå enhetene i det annonserte området via den, selv om Tailscale ikke er installert der. Dette er måten å gjøre en NAS, en skriver eller et administrasjonsgrensesnitt tilgjengelig på uten å sette opp en VPN-klient på hver enkelt enhet.

Området annonseres på ruternoden:

```powershell
tailscale set --advertise-routes=192.168.1.0/24
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `--advertise-routes=<CIDR>` | Annonserer ett eller flere IP-områder (atskilt med komma) som denne noden videresender |
| `--snat-subnet-routes=false` | Videresender uten kilde-NAT, slik at målenhetene ser den faktiske Tailscale-kildeadressen; krever en returrute i det lokale nettverket |
| `--advertise-exit-node` | Kortform som annonserer `0.0.0.0/0` og `::/0`, og dermed tilbyr noden som Exit-Node |

</details>

Trafikken flyter først etter at ruten er **godkjent** i Tailscale-administrasjonen. Det holder ikke bare å annonsere den, og dette er den vanligste feilen: Ruten vises først i rutingtabellen til enhetene som godtar den etter godkjenningen.

## Exit-Node: all trafikk gjennom en node

En Exit-Node annonserer standardruten (`0.0.0.0/0` og `::/0`). Når en enhet velger denne Exit-Node, går all dens **utgående** internettrafikk gjennom noden, ikke bare trafikken til et bestemt nettverk. Dette er nyttig for å gå på internett via et sted med fast IP-adresse eller for å lede trafikken gjennom en pålitelig utgang i et usikkert nettverk.

Forskjellen fra subnettruten er valget på klientsiden: En subnettrute brukes automatisk så snart enheten godtar ruten og kontakter et mål i dette området. En Exit-Node må derimot velges aktivt, og gjelder da for all trafikk:

```powershell
tailscale set --exit-node=100.100.10.10 --exit-node-allow-lan-access
```

<details class="options-details">
<summary>Forklaring av alternativer</summary>

| Alternativ | Virkning |
|---|---|
| `--exit-node=<IP oder Name>` | Velger en Exit-Node; tom (`--exit-node=`) slår den av igjen |
| `--exit-node-allow-lan-access` | Gir fortsatt tilgang til eget lokalt nettverk selv med aktiv Exit-Node |

</details>

Derfor var det feil i supporthverdagen å krysse av for Exit-Node for tilgang til én enkelt NAS: Det ville ha omdirigert all egen trafikk gjennom den fremmede datamaskinen, i stedet for å åpne bare det ene området.

## Sammenligning

| Egenskap | Subnettruter | Exit-Node |
|---|---|---|
| Annonsert rute | Målrettede områder, f.eks. `192.168.1.0/24` | Standardrute `0.0.0.0/0`, `::/0` |
| Klientbruk | Automatisk for mål i området | Må velges aktivt som Exit-Node |
| Omfang | Bare de annonserte nettverkene | All internettrafikk |
| Godkjenning i administrasjonen | Per subnett | Separat som Exit-Node |
| Typisk formål | Gjøre interne tjenester tilgjengelige | Lede utgående trafikk via et sted |

## Hvordan Tailscale implementerer dette teknisk

Begge driftsmodusene bygger på samme grunnlag. Det er nyttig å skille mellom lagene.

**Dataplan via WireGuard.** Hver node har et WireGuard-nøkkelpar. Selve trafikken mellom to noder går direkte som krypterte WireGuard-pakker over UDP, der det er mulig peer-to-peer etter NAT Traversal, ellers via en DERP-reléserver som reservevei. Tailscale finner ikke opp kryptering på nytt, men bruker WireGuard som transport.

**Kontrollplan via Coordination Server.** En sentral Coordination Server distribuerer de offentlige nøklene og et nettverkskart som angir hvilken node som har hvilke adresser og ruter. Coordination Server ser metadataene (hvem som kan snakke med hvem, hvilke ruter som er godkjent), men ikke innholdet i WireGuard-pakkene. Når du annonserer en rute, melder noden dette til kontrollplanet; først med godkjenningen blir ruten en del av nettverkskartet som alle noder mottar.

**På ruternoden.** For at en node skal videresende trafikk for andre enheter, må IP-videresending være aktivert og pakkene formidles mellom Tailscale-grensesnittet og det lokale nettverket. Som standard maskerer Tailscale den videresendte trafikken med kilde-NAT (SNAT): Målenhetene i det lokale nettverket ser den lokale adressen til ruternoden som avsender, ikke Tailscale-adressen til enheten som får tilgang. Dette er det enkle tilfellet, fordi svarpakkene da automatisk finner tilbake til ruteren. Slår du av SNAT, ser målenhetene den faktiske Tailscale-kildeadressen, men da må det lokale nettverket vite hvordan Tailscale-området skal rutes tilbake til ruteren.

**På klientsiden.** En enhet bruker bare andres ruter hvis den godtar dem. På de grafiske klientene for Windows og macOS er det forhåndsinnstilt å godta subnettruter, mens det under Linux aktiveres med `--accept-routes`. Når klienten godtar en rute, legger den den inn i rutingtabellen sin og peker den mot Tailscale-grensesnittet. Pakker til et mål i dette området pakkes deretter inn i WireGuard og sendes til ruternoden. Med Exit-Node er mekanismen den samme, men her peker standardruten mot Exit-Node, og derfor går all trafikk gjennom den.

**Godkjenningen.** At ruter først får virkning etter godkjenning, er en sikkerhetsfunksjon, ikke en omvei: En vilkårlig node skal ikke uten videre kunne trekke trafikk for hele nettverk til seg. Godkjenning kan gjøres manuelt i administrasjonen eller automatisk via `autoApprovers` i tilgangsreglene (ACL-er). Exit-Node og subnettruter godkjennes separat.

## Begrensninger

Begge variantene har begrensninger som påvirker valget:

- **Ruternoden er en flaskehals og et enkelt feilpunkt.** All trafikk til det annonserte nettverket går gjennom denne ene noden, dens WireGuard-kryptering og tilkobling. For feiltoleranse kan flere noder annonsere samme rute; Tailscale bruker da én av dem og bytter ved feil.
- **SNAT skjuler kilden.** Med forhåndsinnstilt kilde-NAT vises alle tilganger med ruternodens adresse. For logging eller tilgangsregler på målenhetene som trenger den faktiske kilden, må du slå av SNAT og sette opp returruten i det lokale nettverket.
- **En Exit-Node videresender virkelig alt.** All trafikk går gjennom noden, med tilsvarende konsekvenser for gjennomstrømning, forsinkelse og konfidensialitet. Operatøren av Exit-Node ser trafikken der den forlater tailnettet. Bruk kun noder du stoler på som Exit-Node.
- **Overlappende subnett er et problem.** Hvis to steder annonserer samme private område, for eksempel `192.168.1.0/24`, kan en klient ikke skille dem fra hverandre. Tailscale tilbyr derfor en omskriving via IPv6 (`4via6`) som gjør områdene entydige.
- **Utløpende nøkler stopper videresendingen.** Hvis nøkkelen til ruternoden utløper, er hele nettverket bak den ikke lenger tilgjengelig. For en permanent ruternode deaktiverer du nøkkelutløp i administrasjonen.

For målrettet tilgang til interne tjenester er subnettruteren nesten alltid det riktige valget: Den åpner bare det som er nødvendig. Velg Exit-Node når du bevisst vil lede all utgående trafikk via et bestemt sted.

## Endringslogg

- **25. oktober 2026:** Kildelenker oppdatert til Tailscales nye adresser.

## Kilder

1.  [Tailscale: Subnet routers](https://tailscale.com/docs/features/subnet-routers): Annonsering av ruter, godkjenning, SNAT-atferd og høy tilgjengelighet med flere rutere.

2.  [Tailscale: Exit nodes](https://tailscale.com/docs/features/exit-nodes): Annonsering av standardrute, valg på klienten og tilgang til eget lokalt nettverk.

3.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): Samspillet mellom WireGuard-dataplan, Coordination Server og DERP-reléer.

4.  [WireGuard: Protokolloversikt](https://www.wireguard.com/protocol/): Det kryptografiske grunnlaget for dataplanet som Tailscale bruker som transport.
