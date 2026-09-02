---
title: "Tailscale: Jämförelse mellan exit-nod och subnätsrutter, och hur de fungerar tekniskt"
navTitle: "Exit-nod vs. subnät"
description: "Exit-noder och subnätsroutrar är två besläktade men olika driftlägen i Tailscale. En subnätsrouter öppnar specifika IP-intervall, medan en exit-nod dirigerar all internettrafik via sig själv. Vad skillnaden innebär i praktiken, hur Tailscale implementerar detta via WireGuard, ruttgodkännande och SNAT, samt var gränserna för respektive variant går."
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
aiPrompt: |
  Du bist mein Netzwerkassistent. Erkläre mir den Unterschied zwischen einem Tailscale-Subnetz-Router und einem Exit-Node, wann ich welchen brauche, und wie Tailscale das technisch umsetzt (WireGuard-Data-Plane, Routen-Freigabe über den Coordination Server, IP-Weiterleitung und SNAT auf dem Router-Node). Hilf mir, die richtige Variante zu wählen und einzurichten.
translationOf: tailscale-exit-node-subnet-routes
url: https://rafaelpfister.ch/sv/blog/tailscale-jamforelse-mellan-exit-nod-och-subnatsrutter-och-hur-de-fungerar-tekniskt
translationSourceHash: f05a193f13dd2b8aba3c9d049ea1c0a1fcc25b12c420a1d520f99854b7883a79
translationModel: gpt-5.6-terra
translatedAt: 2026-09-02T08:01:38.129Z
translationReview: automatic
---

# Tailscale: Jämförelse mellan exit-nod och subnätsrutter, och hur de fungerar tekniskt

En Tailscale-nod är till en början bara sig själv: nåbar via sin Tailscale-adress, men inget annat. För att en nod ska ge andra enheter åtkomst till mer än bara sig själv finns två driftlägen som ofta förväxlas: **subnätsroutern** och **exit-noden**. Båda utökar en nods räckvidd, men i olika riktningar. Den som känner till skillnaden väljer rätt variant och undviker att av misstag leda all trafik via en främmande dator.

Kortversionen: En subnätsrouter öppnar **specifika IP-intervall** bakom noden, till exempel det lokala nätet med en NAS och en skrivare. En exit-nod leder en enhets **hela internettrafik** via sig själv, precis som ett klassiskt full-tunnel-VPN. Båda bygger tekniskt på samma mekanism: annonsering av rutter. Exit-noden är i grunden ett specialfall av subnätsroutern, där standardrutten annonseras.

## Subnätsrouter: riktad åtkomst till ett nät

En subnätsrouter annonserar ett eller flera IP-intervall som den når i det lokala nätet. Andra enheter i tailnet som accepterar dessa rutter kan via dem nå enheterna i det annonserade intervallet, även om Tailscale inte är installerat där. Det är sättet att göra en NAS, skrivare eller administrationspanel tillgänglig utan att installera en VPN-klient på varje enskild enhet.

Intervallet annonseras på routernoden:

```powershell
tailscale set --advertise-routes=192.168.1.0/24
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `--advertise-routes=<CIDR>` | Annonserar ett eller flera IP-intervall (kommaseparerade) som denna nod vidarebefordrar |
| `--snat-subnet-routes=false` | Vidarebefordrar utan käll-NAT så att målenheterna ser den verkliga Tailscale-källadressen; kräver en returväg i det lokala nätet |
| `--advertise-exit-node` | Kortform som annonserar `0.0.0.0/0` och `::/0`, alltså erbjuder noden som exit-nod |

</details>

Trafiken flödar först efter att rutten har **godkänts** i Tailscale-administrationen. Det räcker inte att bara annonsera den, vilket är det vanligaste felet: Rutten visas först efter godkännandet i routingtabellen på de enheter som accepterar den.

## Exit-nod: all trafik via en nod

En exit-nod annonserar standardrutten (`0.0.0.0/0` och `::/0`). När en enhet väljer denna exit-nod går hela dess **utgående** internettrafik genom noden, inte bara trafiken till ett visst nät. Det är användbart för att gå ut på internet via en plats med fast IP-adress eller för att leda trafiken via en betrodd utgång i ett osäkert nät.

Skillnaden jämfört med en subnätsrutt är valet på klientsidan: En subnätsrutt används automatiskt så snart enheten accepterar rutten och kontaktar ett mål inom intervallet. En exit-nod måste däremot väljas aktivt, och gäller då för all trafik:

```powershell
tailscale set --exit-node=100.100.10.10 --exit-node-allow-lan-access
```

<details class="options-details">
<summary>Förklaring av alternativ</summary>

| Alternativ | Effekt |
|---|---|
| `--exit-node=<IP oder Name>` | Väljer en exit-nod; tomt (`--exit-node=`) stänger av den igen |
| `--exit-node-allow-lan-access` | Tillåter åtkomst till det egna lokala nätet även när en exit-nod är aktiv |

</details>

Just därför var det fel i det dagliga supportarbetet att markera exit-noden för åtkomst till en enskild NAS: Det skulle ha dirigerat om all egen trafik via den främmande datorn i stället för att bara öppna det ena intervallet.

## Jämförelse

| Egenskap | Subnätsrouter | Exit-nod |
|---|---|---|
| Annonserad rutt | Riktade intervall, t.ex. `192.168.1.0/24` | Standardrutt `0.0.0.0/0`, `::/0` |
| Klientanvändning | Automatiskt för mål inom intervallet | Måste aktivt väljas som exit-nod |
| Omfattning | Endast de annonserade näten | All internettrafik |
| Godkännande i administrationen | Per subnät | Separat som exit-nod |
| Typiskt syfte | Göra interna tjänster tillgängliga | Leda utgående trafik via en plats |

## Hur Tailscale implementerar detta tekniskt

Båda driftlägena bygger på samma grund. Det är värt att skilja på nivåerna.

**Dataplan via WireGuard.** Varje nod har ett WireGuard-nyckelpar. Den faktiska trafiken mellan två noder går direkt som krypterade WireGuard-paket över UDP, där det är möjligt peer-to-peer efter NAT-traversering, annars via en DERP-reläserver som reservväg. Tailscale uppfinner inte krypteringen på nytt, utan använder WireGuard som transport.

**Kontrollplan via Coordination Server.** En central Coordination Server distribuerar de offentliga nycklarna och en network map som anger vilken nod som har vilka adresser och rutter. Coordination Server ser metadata (vem som får kommunicera med vem och vilka rutter som är godkända), men inte innehållet i WireGuard-paketen. När du annonserar en rutt meddelar noden detta till kontrollplanet; först efter godkännandet blir rutten en del av den network map som alla noder får.

**På routernoden.** För att en nod ska kunna vidarebefordra trafik för andra enheter måste IP-vidarebefordran vara aktiverad och paketen förmedlas mellan Tailscale-gränssnittet och det lokala nätet. Som standard maskerar Tailscale den vidarebefordrade trafiken med käll-NAT (SNAT): Målenheterna i det lokala nätet ser routernodens lokala adress som avsändare, inte Tailscale-adressen för den åtkommande enheten. Det är det enkla fallet eftersom svarspaketen då automatiskt hittar tillbaka till routern. Om du stänger av SNAT ser målenheterna den verkliga Tailscale-källadressen, men då måste det lokala nätet veta hur Tailscale-intervallet ska routas tillbaka till routern.

**På klientsidan.** En enhet använder bara främmande rutter om den accepterar dem. På de grafiska klienterna för Windows och macOS är accepterande av subnätsrutter förinställt, under Linux aktiveras det med `--accept-routes`. När klienten accepterar en rutt lägger den till den i sin routingtabell och pekar den mot Tailscale-gränssnittet. Paket till ett mål inom detta intervall paketeras sedan i WireGuard och skickas till routernoden. För exit-noden är mekanismen densamma, men här pekar standardrutten mot exit-noden, vilket gör att all trafik går genom den.

**Godkännandet.** Att rutter får effekt först efter godkännande är en säkerhetsfunktion, inte en omväg: En godtycklig nod ska inte oombedd kunna dra till sig trafik för hela nät. Godkännande kan göras manuellt i administrationen eller automatiskt via `autoApprovers` i åtkomstreglerna (ACL:er). Exit-noder och subnätsrutter godkänns separat.

## Begränsningar

Båda varianterna har begränsningar som påverkar valet:

- **Routernoden är en flaskhals och en single point of failure.** All trafik för det annonserade nätet går via denna enda nod, dess WireGuard-kryptering och dess anslutning. För feltolerans kan flera noder annonsera samma rutt; Tailscale använder då en av dem och växlar vid fel.
- **SNAT döljer källan.** Med den förinställda käll-NAT:en visas all åtkomst under routernodens adress. För loggning eller åtkomstregler på målenheterna som behöver den verkliga källan måste du stänga av SNAT och konfigurera returvägen i det lokala nätet.
- **En exit-nod leder verkligen allt.** All trafik går via noden, med motsvarande konsekvenser för genomströmning, latens och konfidentialitet. Operatören av exit-noden ser trafiken vid den punkt där den lämnar tailnet. Använd endast noder som du litar på som exit-noder.
- **Överlappande subnät är ett problem.** Om två platser annonserar samma privata intervall, till exempel `192.168.1.0/24`, kan en klient inte skilja dem åt. Tailscale erbjuder för detta en omskrivning via IPv6 (`4via6`), som gör intervallen entydiga.
- **Utgående nycklar stoppar vidarebefordringen.** Om routernodens nyckel löper ut är hela nätet bakom den inte längre nåbart. För en permanent routernod inaktiverar du nyckelutgången i administrationen.

För riktad åtkomst till interna tjänster är subnätsroutern nästan alltid rätt val: Den öppnar bara det som behövs. Välj exit-noden när du medvetet vill leda all utgående trafik via en viss plats.

## Källor

1.  [Tailscale: Subnet routers](https://tailscale.com/kb/1019/subnets): Annonsering av rutter, godkännande, SNAT-beteende och hög tillgänglighet med flera routrar.

2.  [Tailscale: Exit nodes](https://tailscale.com/kb/1103/exit-nodes): Annonsering av standardrutt, val på klienten och åtkomst till det egna lokala nätet.

3.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): Samspelet mellan WireGuard-dataplan, Coordination Server och DERP-reläer.

4.  [WireGuard: Protokollöversikt](https://www.wireguard.com/protocol/): Den kryptografiska grunden för dataplanet som Tailscale använder som transport.
