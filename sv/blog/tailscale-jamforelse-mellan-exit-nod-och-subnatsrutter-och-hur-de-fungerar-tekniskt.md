---
title: "Tailscale: Jämförelse av Exit Node och subnätsrutter – och hur de fungerar tekniskt"
navTitle: "Exit Node vs. subnät"
description: "Exit Node och subnätsroutrar är två relaterade men olika driftlägen i Tailscale. En subnätsrouter öppnar specifika IP-intervall, medan en Exit Node dirigerar all internettrafik via sig själv. Här förklaras den praktiska skillnaden, hur Tailscale implementerar detta via WireGuard, ruttgodkännande och SNAT, samt begränsningarna för respektive variant."
date: "2026-09-02"
kategorie: "Nätverk och VPN"
timeToRead: "11 min läsning"
themen:
  - tailscale
produkte:
  - "tailscale"
protokolle:
  - "tcp"
  - "haertung"
slug: "tailscale-jamforelse-mellan-exit-nod-och-subnatsrutter-och-hur-de-fungerar-tekniskt"
translationId: "article-c26cca4d635b9a04"
translationOf: tailscale-exit-node-subnet-routes
translationSourceHash: cefcefd45f617dfb668890986aaad256dbeca2b84d7c3472cd9eb5d6bb049911
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:53:49.601Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/sv/blog/tailscale-jamforelse-mellan-exit-nod-och-subnatsrutter-och-hur-de-fungerar-tekniskt
---

# Tailscale: Jämförelse av Exit Node och subnätsrutter – och hur de fungerar tekniskt

En Tailscale-nod är till en början bara sig själv: nåbar via sin Tailscale-adress, men inget mer. För att en nod ska kunna ge andra enheter åtkomst till mer än bara sig själv finns två driftlägen som ofta förväxlas: **subnätsroutern** och **Exit Node**. Båda utökar en nods räckvidd, men åt olika håll. Den som känner till skillnaden väljer rätt variant och undviker att av misstag dirigera all trafik via en främmande dator.

Kortversionen: En subnätsrouter öppnar **specifika IP-intervall** bakom noden, till exempel det lokala nätet med en NAS och en skrivare. En Exit Node dirigerar en enhets **hela internettrafik** via sig själv, precis som ett klassiskt full tunnel-VPN. Båda bygger tekniskt på samma mekanism: att annonsera rutter. Exit Node är i grunden ett specialfall av subnätsroutern, där standardrutten annonseras.

## Subnätsrouter: riktad åtkomst till ett nät

En subnätsrouter annonserar ett eller flera IP-intervall som den når i det lokala nätet. Andra enheter i tailnet som accepterar dessa rutter kan via dem nå enheterna i det annonserade intervallet, även om Tailscale inte är installerat där. Det är sättet att göra en NAS, en skrivare eller ett administrationsgränssnitt åtkomligt utan att installera en VPN-klient på varje enskild enhet.

Intervallet annonseras på routernoden:

```powershell
tailscale set --advertise-routes=192.168.1.0/24
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `--advertise-routes=<CIDR>` | Annonserar ett eller flera IP-intervall (kommaseparerade) som denna nod vidarebefordrar |
| `--snat-subnet-routes=false` | Vidarebefordrar utan käll-NAT så att målenheterna ser den verkliga Tailscale-källadressen; kräver en returroute i det lokala nätet |
| `--advertise-exit-node` | Kortform som annonserar `0.0.0.0/0` och `::/0`, alltså erbjuder noden som Exit Node |

</details>

Trafiken går först efter att rutten har **godkänts** i Tailscale-administrationen. Det räcker inte att bara annonsera den, och detta är det vanligaste felet: Rutten visas först efter godkännandet i routingtabellen på de enheter som accepterar den.

## Exit Node: all trafik via en nod

En Exit Node annonserar standardrutten (`0.0.0.0/0` och `::/0`). Om en enhet väljer denna Exit Node går dess **hela** utgående internettrafik genom noden, inte bara trafik till ett visst nät. Det är användbart för att gå ut på internet via en plats med fast IP-adress eller för att dirigera trafik via en betrodd utgång i ett osäkert nät.

Skillnaden jämfört med en subnätsrutt är valet på klientsidan: En subnätsrutt används automatiskt så snart enheten accepterar rutten och kontaktar ett mål inom intervallet. En Exit Node måste däremot väljas aktivt, och gäller då för all trafik:

```powershell
tailscale set --exit-node=100.100.10.10 --exit-node-allow-lan-access
```

<details class="options-details">
<summary>Alternativ förklarade</summary>

| Alternativ | Effekt |
|---|---|
| `--exit-node=<IP oder Name>` | Väljer en Exit Node; tomt (`--exit-node=`) stänger av den igen |
| `--exit-node-allow-lan-access` | Tillåter åtkomst till det egna lokala nätet trots aktiv Exit Node |

</details>

Just därför var det fel i supportarbetet att välja Exit Node för åtkomst till en enskild NAS: Det hade dirigerat om hela den egna trafiken via den främmande datorn, i stället för att bara öppna det enda intervallet.

## Jämförelse

| Egenskap | Subnätsrouter | Exit Node |
|---|---|---|
| Annonserad rutt | Specifika intervall, t.ex. `192.168.1.0/24` | Standardrutt `0.0.0.0/0`, `::/0` |
| Klientanvändning | Automatiskt för mål inom intervallet | Måste aktivt väljas som Exit Node |
| Omfattning | Endast de annonserade näten | Hela internettrafiken |
| Godkännande i administrationen | Per subnät | Separat som Exit Node |
| Typiskt syfte | Göra interna tjänster åtkomliga | Dirigera utgående trafik via en plats |

## Hur Tailscale implementerar detta tekniskt

Båda driftlägena bygger på samma grund. Det är värt att skilja mellan lagren.

**Dataplan via WireGuard.** Varje nod har ett WireGuard-nyckelpar. Den faktiska trafiken mellan två noder går direkt som krypterade WireGuard-paket via UDP, där det är möjligt peer-to-peer efter NAT-traversering, annars via en DERP-reläserver som reservväg. Tailscale uppfinner inte krypteringen på nytt, utan använder WireGuard som transport.

**Kontrollplan via Coordination Server.** En central Coordination Server distribuerar de offentliga nycklarna och en network map som anger vilken nod som har vilka adresser och rutter. Coordination Server ser metadata (vem som får kommunicera med vem och vilka rutter som är godkända), men inte innehållet i WireGuard-paketen. När du annonserar en rutt rapporterar noden detta till Control Plane; först efter godkännande blir rutten en del av network map som alla noder får.

**På routernoden.** För att en nod ska kunna vidarebefordra trafik åt andra enheter måste IP-vidarebefordran vara aktiverad och paketen förmedlas mellan Tailscale-gränssnittet och det lokala nätet. Som standard maskerar Tailscale den vidarebefordrade trafiken med käll-NAT (SNAT): Målenheterna i det lokala nätet ser routernodens lokala adress som avsändare, inte Tailscale-adressen för den åtkommande enheten. Det är det enkla fallet, eftersom svarspaketen då automatiskt hittar tillbaka till routern. Om du stänger av SNAT ser målenheterna den verkliga Tailscale-källadressen, men då måste det lokala nätet veta hur Tailscale-intervallet ska routas tillbaka till routern.

**På klientsidan.** En enhet använder främmande rutter endast om den accepterar dem. På de grafiska klienterna för Windows och macOS är acceptans av subnätsrutter förinställt, medan det aktiveras under Linux med `--accept-routes`. När klienten accepterar en rutt lägger den in den i sin routingtabell och pekar den mot Tailscale-gränssnittet. Paket till ett mål i detta intervall kapslas då in i WireGuard och skickas till routernoden. För Exit Node är det samma mekanism, men här pekar standardrutten på Exit Node, vilket innebär att all trafik går genom den.

**Godkännandet.** Att rutter får effekt först efter godkännande är en säkerhetsfunktion, inte en omväg: En godtycklig nod ska inte utan vidare kunna dra till sig trafik för hela nät. Godkännande kan göras manuellt i administrationen eller automatiskt via `autoApprovers` i åtkomstreglerna (ACL:er). Exit Node och subnätsrutter godkänns separat.

## Begränsningar

Båda varianterna har begränsningar som påverkar valet:

- **Routernoden är en flaskhals och en single point of failure.** All trafik för det annonserade nätet går genom denna enda nod, dess WireGuard-kryptering och dess anslutning. För feltolerans kan flera noder annonsera samma rutt; Tailscale använder då en av dem och växlar vid fel.
- **SNAT döljer källan.** Med det förinställda käll-NAT visas all åtkomst under routernodens adress. För loggning eller åtkomstregler på målenheterna som behöver den verkliga källan måste du stänga av SNAT och konfigurera returrouetten i det lokala nätet.
- **En Exit Node vidarebefordrar verkligen allt.** All trafik går genom noden, med motsvarande konsekvenser för genomströmning, latens och konfidentialitet. Operatören av Exit Node ser trafiken där den lämnar tailnet. Använd endast noder som du litar på som Exit Node.
- **Överlappande subnät är ett problem.** Om två platser annonserar samma privata intervall, till exempel `192.168.1.0/24`, kan en klient inte skilja dem åt. Tailscale erbjuder för detta en omskrivning via IPv6 (`4via6`), som gör intervallen entydiga.
- **Utgående nycklar stoppar vidarebefordran.** Om routernodens nyckel löper ut är hela nätet bakom den inte längre åtkomligt. För en permanent routernod inaktiverar du nyckelutgången i administrationen.

För riktad åtkomst till interna tjänster är subnätsroutern nästan alltid rätt val: Den öppnar endast det som behövs. Välj Exit Node när du medvetet vill leda all utgående trafik via en viss plats.

## Ändringshistorik

- **25 oktober 2026:** Källänkar uppdaterade till Tailscales nya adresser.

## Källor

1.  [Tailscale: Subnet routers](https://tailscale.com/docs/features/subnet-routers): annonsera rutter, godkännande, SNAT-beteende och hög tillgänglighet med flera routrar.

2.  [Tailscale: Exit nodes](https://tailscale.com/docs/features/exit-nodes): annonsera standardrutt, val på klienten och åtkomst till det egna lokala nätet.

3.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): samspelet mellan WireGuard-dataplanet, Coordination Server och DERP-reläer.

4.  [WireGuard: Protokollöversikt](https://www.wireguard.com/protocol/): den kryptografiska grunden för dataplanet som Tailscale använder som transport.
