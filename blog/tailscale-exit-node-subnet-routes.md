---
title: "Tailscale: Exit-Node und Subnetz-Routen im Vergleich, und wie sie technisch funktionieren"
navTitle: "Exit-Node vs. Subnetz"
description: "Exit-Node und Subnetz-Router sind bei Tailscale zwei verwandte, aber unterschiedliche Betriebsarten. Ein Subnetz-Router öffnet gezielt bestimmte IP-Bereiche, ein Exit-Node leitet den gesamten Internetverkehr über sich. Was der Unterschied praktisch bedeutet, wie Tailscale das über WireGuard, die Routen-Freigabe und SNAT umsetzt, und wo die Grenzen jeder Variante liegen."
date: "2026-09-02"
kategorie: "Netzwerk und VPN"
timeToRead: "11 Min. Lesezeit"
themen:
  - "tailscale"
produkte:
  - "tailscale"
protokolle:
  - "tcp"
  - "haertung"
slug: "tailscale-exit-node-subnet-routes"
translationId: "article-c26cca4d635b9a04"
url: "https://rafaelpfister.ch/blog/tailscale-exit-node-subnet-routes"
aiPrompt: |
  Du bist mein Netzwerkassistent. Erkläre mir den Unterschied zwischen einem Tailscale-Subnetz-Router und einem Exit-Node, wann ich welchen brauche, und wie Tailscale das technisch umsetzt (WireGuard-Data-Plane, Routen-Freigabe über den Coordination Server, IP-Weiterleitung und SNAT auf dem Router-Node). Hilf mir, die richtige Variante zu wählen und einzurichten.
---
# Tailscale: Exit-Node und Subnetz-Routen im Vergleich, und wie sie technisch funktionieren

Ein Tailscale-Node ist zunächst nur er selbst: erreichbar über seine Tailscale-Adresse, sonst nichts. Damit ein Node anderen Geräten Zugang zu mehr als sich selbst verschafft, gibt es zwei Betriebsarten, die oft verwechselt werden: den **Subnetz-Router** und den **Exit-Node**. Beide erweitern die Reichweite eines Node, aber in unterschiedliche Richtungen. Wer den Unterschied kennt, wählt die passende Variante und vermeidet, versehentlich den gesamten Verkehr über einen fremden Rechner zu leiten.

Die kurze Fassung: Ein Subnetz-Router öffnet **gezielt bestimmte IP-Bereiche** hinter dem Node, etwa das lokale Netz mit einem NAS und einem Drucker. Ein Exit-Node leitet den **gesamten Internetverkehr** eines Geräts über sich, so wie ein klassisches Full-Tunnel-VPN. Beide beruhen technisch auf demselben Mechanismus: dem Advertisen von Routen. Der Exit-Node ist im Grunde ein Sonderfall des Subnetz-Routers, bei dem die Standardroute advertised wird.

## Subnetz-Router: gezielter Zugang zu einem Netz

Ein Subnetz-Router advertised einen oder mehrere IP-Bereiche, die er im lokalen Netz erreicht. Andere Geräte im Tailnet, die diese Routen annehmen, erreichen darüber die Geräte im advertiseten Bereich, auch wenn dort kein Tailscale installiert ist. Das ist der Weg, um ein NAS, einen Drucker oder eine Verwaltungsoberfläche erreichbar zu machen, ohne auf jedem einzelnen Gerät einen VPN-Client einzurichten.

Advertised wird der Bereich auf dem Router-Node:

```powershell
tailscale set --advertise-routes=192.168.1.0/24
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `--advertise-routes=<CIDR>` | Advertised einen oder mehrere IP-Bereiche (durch Komma getrennt), die dieser Node weiterleitet |
| `--snat-subnet-routes=false` | Leitet ohne Quell-NAT weiter, damit die Zielgeräte die echte Tailscale-Quelladresse sehen; verlangt eine Rückroute im lokalen Netz |
| `--advertise-exit-node` | Kurzform, die `0.0.0.0/0` und `::/0` advertised, also den Node als Exit-Node anbietet |

</details>

Der Verkehr fliesst nur, nachdem die Route in der Tailscale-Verwaltung **freigegeben** wurde. Ein reines Advertisen genügt nicht, das ist der häufigste Fehler: Die Route erscheint erst nach der Freigabe in der Routing-Tabelle der annehmenden Geräte.

## Exit-Node: der gesamte Verkehr über einen Node

Ein Exit-Node advertised die Standardroute (`0.0.0.0/0` und `::/0`). Wählt ein Gerät diesen Exit-Node aus, läuft sein **gesamter** ausgehender Internetverkehr durch den Node, nicht nur der Verkehr zu einem bestimmten Netz. Das ist nützlich, um über einen Standort mit fester IP ins Internet zu gehen oder um in einem unsicheren Netz den Verkehr über einen vertrauenswürdigen Ausgang zu leiten.

Der Unterschied zur Subnetz-Route ist die Auswahl auf der Client-Seite: Eine Subnetz-Route wird automatisch genutzt, sobald das Gerät die Route annimmt und ein Ziel in diesem Bereich anspricht. Ein Exit-Node dagegen muss aktiv ausgewählt werden, und dann gilt er für allen Verkehr:

```powershell
tailscale set --exit-node=100.100.10.10 --exit-node-allow-lan-access
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `--exit-node=<IP oder Name>` | Wählt einen Exit-Node; leer (`--exit-node=`) schaltet ihn wieder ab |
| `--exit-node-allow-lan-access` | Lässt trotz aktivem Exit-Node den Zugriff auf das eigene lokale Netz zu |

</details>

Genau darum war es im Support-Alltag falsch, für den Zugang zu einem einzelnen NAS den Exit-Node anzuhaken: Das hätte den kompletten eigenen Verkehr über den fremden Rechner umgeleitet, statt nur den einen Bereich zu öffnen.

## Gegenüberstellung

| Eigenschaft | Subnetz-Router | Exit-Node |
|---|---|---|
| Advertisete Route | Gezielte Bereiche, z. B. `192.168.1.0/24` | Standardroute `0.0.0.0/0`, `::/0` |
| Client-Nutzung | Automatisch für Ziele im Bereich | Muss aktiv als Exit-Node gewählt werden |
| Umfang | Nur die advertiseten Netze | Der gesamte Internetverkehr |
| Freigabe in der Verwaltung | Pro Subnetz | Separat als Exit-Node |
| Typischer Zweck | Interne Dienste erreichbar machen | Ausgehenden Verkehr über einen Standort leiten |

## Wie Tailscale das technisch umsetzt

Beide Betriebsarten laufen auf derselben Grundlage. Es lohnt sich, die Ebenen zu trennen.

**Data Plane über WireGuard.** Jeder Node hat ein WireGuard-Schlüsselpaar. Der eigentliche Verkehr zwischen zwei Nodes läuft direkt als verschlüsselte WireGuard-Pakete über UDP, wo möglich Peer-zu-Peer nach NAT Traversal, sonst über einen DERP-Relay-Server als Rückfallweg. Tailscale erfindet die Verschlüsselung nicht neu, sondern nutzt WireGuard als Transport.

**Control Plane über den Coordination Server.** Ein zentraler Coordination Server verteilt die öffentlichen Schlüssel und eine network map, die festhält, welcher Node welche Adressen und Routen besitzt. Der Coordination Server sieht die Metadaten (wer mit wem sprechen darf, welche Routen freigegeben sind), aber nicht den Inhalt der WireGuard-Pakete. Wenn Sie eine Route advertisen, meldet der Node das an die Control Plane; erst mit der Freigabe wird die Route Teil der network map, die alle Nodes erhalten.

**Auf dem Router-Node.** Damit ein Node Verkehr für andere Geräte weiterleitet, muss er IP-Weiterleitung aktiviert haben und die Pakete zwischen der Tailscale-Schnittstelle und dem lokalen Netz vermitteln. Standardmässig maskiert Tailscale den weitergeleiteten Verkehr mit Quell-NAT (SNAT): Die Zielgeräte im lokalen Netz sehen als Absender die lokale Adresse des Router-Node, nicht die Tailscale-Adresse des zugreifenden Geräts. Das ist der einfache Fall, weil die Antwortpakete so von selbst zum Router zurückfinden. Schalten Sie SNAT ab, sehen die Zielgeräte die echte Tailscale-Quelladresse, aber dann muss das lokale Netz wissen, wie es den Tailscale-Bereich zum Router zurückroutet.

**Auf der Client-Seite.** Ein Gerät nutzt fremde Routen nur, wenn es sie annimmt. Auf den grafischen Clients für Windows und macOS ist das Annehmen von Subnetz-Routen voreingestellt, unter Linux wird es mit `--accept-routes` eingeschaltet. Nimmt der Client eine Route an, trägt er sie in seine Routing-Tabelle ein und zeigt auf die Tailscale-Schnittstelle. Pakete an ein Ziel in diesem Bereich werden dann in WireGuard verpackt und zum Router-Node geschickt. Beim Exit-Node ist es dieselbe Mechanik, nur zeigt hier die Standardroute auf den Exit-Node, weshalb aller Verkehr durch ihn läuft.

**Die Freigabe.** Dass Routen erst nach der Freigabe wirken, ist ein Sicherheitsmerkmal, kein Umweg: Ein beliebiger Node soll nicht ungefragt Verkehr für ganze Netze an sich ziehen können. Freigeben lässt sich manuell in der Verwaltung oder automatisch über `autoApprovers` in den Zugriffsregeln (ACLs). Exit-Node und Subnetz-Routen werden dabei getrennt freigegeben.

## Grenzen

Beide Varianten haben Grenzen, die die Wahl beeinflussen:

- **Der Router-Node ist ein Engpass und ein Single Point of Failure.** Aller Verkehr für das advertisete Netz läuft über diesen einen Node, über seine WireGuard-Verschlüsselung und seine Anbindung. Für Ausfallsicherheit können mehrere Nodes dieselbe Route advertisen; Tailscale nutzt dann einen davon und wechselt bei Ausfall.
- **SNAT verdeckt die Quelle.** Mit dem voreingestellten Quell-NAT erscheinen alle Zugriffe unter der Adresse des Router-Node. Für Protokollierung oder Zugriffsregeln auf den Zielgeräten, die die echte Quelle brauchen, müssen Sie SNAT abschalten und die Rückroute im lokalen Netz einrichten.
- **Ein Exit-Node leitet wirklich alles.** Der gesamte Verkehr läuft über den Node, mit den entsprechenden Folgen für Durchsatz, Latenz und Vertraulichkeit. Der Betreiber des Exit-Node sieht den Verkehr an dem Punkt, an dem er das Tailnet verlässt. Nutzen Sie als Exit-Node nur Nodes, denen Sie vertrauen.
- **Überlappende Subnetze sind ein Problem.** Advertisen zwei Standorte denselben privaten Bereich, etwa `192.168.1.0/24`, kann ein Client sie nicht auseinanderhalten. Tailscale bietet dafür eine Umschreibung über IPv6 (`4via6`), die die Bereiche eindeutig macht.
- **Ablaufende Schlüssel legen die Weiterleitung still.** Läuft der Schlüssel des Router-Node ab, ist das ganze dahinterliegende Netz nicht mehr erreichbar. Für einen dauerhaften Router-Node deaktivieren Sie den Schlüsselablauf in der Verwaltung.

Für den gezielten Zugriff auf interne Dienste ist der Subnetz-Router fast immer die richtige Wahl: Er öffnet nur, was nötig ist. Den Exit-Node nehmen Sie, wenn Sie bewusst den gesamten ausgehenden Verkehr über einen bestimmten Standort führen wollen.

## Quellen

1.  [Tailscale: Subnet routers](https://tailscale.com/kb/1019/subnets): Routen advertisen, Freigabe, SNAT-Verhalten und Hochverfügbarkeit mit mehreren Routern.

2.  [Tailscale: Exit nodes](https://tailscale.com/kb/1103/exit-nodes): Standardroute advertisen, Auswahl auf dem Client und der Zugriff auf das eigene lokale Netz.

3.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): Zusammenspiel von WireGuard-Data-Plane, Coordination Server und DERP-Relays.

4.  [WireGuard: Protokollüberblick](https://www.wireguard.com/protocol/): Die kryptografische Grundlage der Data Plane, die Tailscale als Transport nutzt.
