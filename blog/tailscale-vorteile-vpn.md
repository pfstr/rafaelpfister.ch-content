---
title: "Was ist eigentlich Tailscale und welche Vorteile bietet es gegenüber traditionellen VPN-Verbindungen?"
navTitle: "Tailscale vs. VPN"
description: "Tailscale baut auf WireGuard ein Mesh-VPN auf, in dem sich Geräte direkt verbinden statt über einen zentralen VPN-Konzentrator. Wie Coordination Server, NAT-Traversal und DERP-Relays zusammenspielen, worin die Vorteile gegenüber IPsec- und SSL-VPN liegen und welche Abhängigkeiten und Grenzen Sie vor dem Einsatz kennen sollten."
date: "2026-10-01"
kategorie: "VPN und Fernzugriff"
timeToRead: "11 Min. Lesezeit"
themen:
  - "vpn-fernzugriff"
produkte:
  - "tailscale"
protokolle:
  - "tcp"
  - "haertung"
slug: "tailscale-vorteile-vpn"
translationId: "article-91fddf1e0239f5c4"
url: "https://rafaelpfister.ch/blog/tailscale-vorteile-vpn"
aiPrompt: |
  Du bist mein Netzwerk-Assistent. Hilf mir einzuschätzen, ob Tailscale unser bestehendes VPN ganz oder teilweise ersetzen kann: Ist-Zustand aufnehmen (VPN-Gateway, Benutzer, Standorte, erreichbare Netze), Zugriffsregeln nach dem Prinzip der minimalen Rechte als Tailscale-Policy entwerfen, Subnet Router und Exit Nodes planen und Abhängigkeiten wie Identity Provider, Datenschutz nach revDSG und Koexistenz mit anderen VPN-Clients prüfen.
---
# Was ist eigentlich Tailscale und welche Vorteile bietet es gegenüber traditionellen VPN-Verbindungen?

Tailscale ist ein VPN-Dienst, der Geräte zu einem privaten Netz verbindet, dem sogenannten Tailnet. Technisch basiert er auf WireGuard. Der Unterschied zu einem klassischen Firmen-VPN liegt in der Topologie: Die Geräte bauen ihre verschlüsselten Tunnel direkt untereinander auf (Mesh). Ein zentraler VPN-Konzentrator, durch den der gesamte Verkehr läuft, entfällt. Zentral bleibt nur die Verwaltung: Ein Coordination Server verteilt öffentliche Schlüssel, Adressen und Zugriffsregeln, sieht aber selbst keine Nutzdaten.

Für den Einstieg genügen ein Konto bei einem Identity Provider (Microsoft, Google, GitHub, Apple oder ein OIDC-Anbieter) und der Client auf jedem Gerät. Portfreigaben in der Firewall sind in der Regel nicht nötig. Damit ist Tailscale sowohl für das Heimnetz als auch für den Fernzugriff auf Server und Kundensysteme verbreitet.

## Wie ein traditionelles VPN funktioniert

Ein klassisches Remote-Access-VPN arbeitet nach dem Hub-and-Spoke-Prinzip. Am Rand des Firmennetzes steht ein VPN-Gateway (Firewall oder Appliance), das aus dem Internet erreichbar ist. Der Client auf dem Notebook baut einen Tunnel zu diesem Gateway auf, gängig sind IPsec/IKEv2 (UDP 500 und 4500), SSL-VPN-Varianten über TCP 443 oder OpenVPN. Nach der Anmeldung erhält das Gerät eine Adresse aus einem Pool und Routen in die internen Netze.

Dieses Modell hat sich über Jahrzehnte bewährt, bringt aber strukturelle Eigenschaften mit, die im heutigen Betrieb Aufwand verursachen:

- **Öffentlich erreichbares Gateway:** Der VPN-Konzentrator muss aus dem Internet erreichbar sein und ist damit ein bevorzugtes Angriffsziel. Schwachstellen in VPN-Appliances wurden in den letzten Jahren wiederholt aktiv ausgenutzt; die US-Behörde CISA ordnete im Januar 2024 mit der Emergency Directive 24-01 sogar die Trennung betroffener Ivanti-Gateways vom Netz an.
- **Single Point of Failure und Engpass:** Sämtlicher Verkehr läuft über das Gateway. Fällt es aus oder ist die Bandbreite erschöpft, betrifft das alle Benutzer gleichzeitig.
- **Umwege (Hairpinning):** Arbeiten zwei Mitarbeitende im Homeoffice auf denselben Server in der Cloud zu, läuft der Verkehr zuerst ins Rechenzentrum und von dort wieder hinaus.
- **Grobe Zugriffsrechte:** Nach dem Verbindungsaufbau steht dem Gerät oft ein ganzes Netzsegment offen. Feingranulare Regeln pro Benutzer und Dienst sind möglich, werden aber selten konsequent gepflegt.
- **Site-to-Site-Aufwand:** Jeder zusätzliche Standort braucht einen eigenen Tunnel mit abgestimmten Parametern (Phase-1/Phase-2-Proposals, Pre-Shared Keys oder Zertifikate, feste öffentliche IP-Adressen).

## Wie Tailscale aufgebaut ist

Tailscale trennt die Control Plane von der Data Plane. Die Control Plane ist der Coordination Server, den Tailscale als Clouddienst betreibt. Die Data Plane sind die WireGuard-Tunnel zwischen den Geräten.

| Komponente | Aufgabe |
|---|---|
| Client (`tailscaled`) | Erzeugt das Schlüsselpaar lokal, baut WireGuard-Tunnel zu den anderen Nodes auf, setzt die Zugriffsregeln lokal durch |
| Coordination Server | Authentifiziert Geräte über den Identity Provider, verteilt öffentliche Schlüssel, Adressen, DNS-Einstellungen und die Policy an alle Nodes |
| DERP-Relays | Leiten verschlüsselte Pakete weiter, wenn keine direkte Verbindung zustande kommt |
| Peer Relays | Eigene Geräte im Tailnet, die als Relay mit höherem Durchsatz dienen; sie werden vor DERP bevorzugt |

Der private Schlüssel eines Geräts verlässt das Gerät nie. Der Coordination Server kennt nur die öffentlichen Schlüssel und kann den Verkehr deshalb nicht entschlüsseln. Jedes Gerät erhält eine feste Adresse aus dem Bereich `100.64.0.0/10` (dem Adressraum für Carrier-Grade NAT) sowie eine IPv6-Adresse aus `fd7a:115c:a1e0::/48`. Über MagicDNS sind die Geräte zusätzlich unter ihrem Namen erreichbar, zum Beispiel `nas` oder `nas.tailnet-name.ts.net`.

### NAT-Traversal: warum keine Portfreigaben nötig sind

Die meisten Geräte stehen hinter einem NAT-Router oder einer Firewall und sind von aussen nicht direkt erreichbar. Tailscale löst das mit NAT-Traversal: Beide Seiten ermitteln über STUN ihre öffentliche Adresse und den zugewiesenen Port, tauschen diese Informationen über den Coordination Server aus und senden gleichzeitig UDP-Pakete aneinander. Die ausgehenden Pakete öffnen auf beiden Firewalls einen Zustandseintrag, über den die Pakete der Gegenseite anschliessend hereinkommen (UDP Hole Punching).

Gelingt das nicht, etwa bei restriktiven Firewalls, die ausgehendes UDP sperren, oder bei bestimmten Formen von Carrier-Grade NAT, läuft der Verkehr über ein DERP-Relay per HTTPS. Auch dort bleibt er Ende-zu-Ende mit WireGuard verschlüsselt; das Relay sieht nur verschlüsselte Pakete. Der Preis ist eine höhere Latenz und ein geringerer Durchsatz. Peer Relays verringern diesen Nachteil, indem ein eigenes Gerät mit guter Anbindung die Weiterleitung übernimmt.

## Die Vorteile gegenüber einem traditionellen VPN

| Kriterium | Traditionelles VPN | Tailscale |
|---|---|---|
| Topologie | Hub-and-Spoke über ein zentrales Gateway | Mesh, direkte Verbindungen zwischen den Geräten |
| Eingehende Ports | Gateway muss aus dem Internet erreichbar sein | Keine eingehenden Portfreigaben nötig |
| Authentifizierung | Lokale Konten, RADIUS, Zertifikate, oft separates MFA | Anmeldung über den bestehenden Identity Provider inklusive dessen MFA |
| Zugriffsrechte | Häufig pro Netzsegment | Pro Benutzer, Gruppe, Gerät und Port in einer zentralen Policy |
| Neuer Standort | Site-to-Site-Tunnel mit abgestimmten Parametern | Client oder Subnet Router installieren |
| Ausfall des Zentrums | Kein Zugriff mehr für alle | Bestehende Verbindungen laufen weiter; neue Geräte und Policy-Änderungen warten |
| Protokoll | IPsec, SSL-VPN, OpenVPN | WireGuard |

### Weniger Angriffsfläche

Da die Clients ihre Verbindungen ausgehend aufbauen, braucht kein Gerät einen offenen Port im Internet. Ein Server, der nur über das Tailnet erreichbar sein soll, kann seine Dienste ausschliesslich an die Tailscale-Schnittstelle binden. Für Port-Scanner im Internet ist er dann nicht sichtbar. WireGuard selbst ist mit rund 4000 Zeilen Kernel-Code deutlich kleiner als typische IPsec- oder SSL-VPN-Implementierungen und verwendet einen festen Satz moderner Verfahren (Curve25519, ChaCha20-Poly1305, BLAKE2s). Eine Aushandlung von Cipher-Suites, wie sie bei IPsec regelmässig zu Fehlkonfigurationen führt, gibt es nicht.

### Identität statt Netzwerkadresse

Jedes Gerät im Tailnet ist an einen Benutzer oder ein Tag gebunden. Die Anmeldung läuft über den Identity Provider, den Sie bereits verwenden; MFA und Conditional Access aus Microsoft Entra ID gelten damit auch für den Netzwerkzugang. Neue Geräte können sich nur nach erfolgreicher Anmeldung beim Identity Provider ins Tailnet einbinden, und jeder Geräteschlüssel läuft standardmässig nach 180 Tagen ab. Verlässt ein Mitarbeitender das Unternehmen, sperren Sie das Konto im Identity Provider; mit SCIM-Provisionierung (ab Tarif Standard) wird der Benutzer im Tailnet automatisch deaktiviert und seine Geräte verlieren den Zugang.

### Zugriffsregeln nach dem Prinzip der minimalen Rechte

Standardmässig darf in einem neuen Tailnet jedes Gerät jedes andere erreichen. Für den produktiven Einsatz legen Sie die Zugriffsrechte in einer zentralen Policy-Datei (HuJSON) fest. Das folgende Beispiel erlaubt der Gruppe der Administratoren SSH und HTTPS auf alle Server mit dem Tag `tag:server`, allen übrigen Benutzern nur HTTPS auf den Intranet-Server:

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
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `groups` | Definiert Benutzergruppen; Mitglieder werden über ihre Anmeldeadresse beim Identity Provider angegeben |
| `tagOwners` | Legt fest, wer Geräten ein Tag zuweisen darf; getaggte Geräte gehören keinem Benutzer, sondern dem Tag |
| `grants` | Liste der erlaubten Verbindungen; was nicht ausdrücklich erlaubt ist, wird blockiert |
| `src` | Quelle der Verbindung: Benutzer, Gruppe, Tag oder `autogroup:member` (alle Benutzer des Tailnets) |
| `dst` | Ziel der Verbindung: Tag, Host-Alias, Gerät oder Subnetz |
| `ip` | Erlaubte Protokolle und Ports, zum Beispiel `tcp:22` oder `*` für alles |
| `hosts` | Alias-Namen für Tailnet-Adressen oder Subnetze, die in den Regeln verwendet werden |

</details>

Die Regeln setzt jeder Client lokal durch. Ein Paket, das die Policy nicht erlaubt, verwirft bereits das Zielgerät. Der Coordination Server verteilt die Policy nur.

### Direkte Verbindungen statt Umwege

Weil die Tunnel direkt zwischen den Geräten bestehen, nimmt der Verkehr den kürzesten Weg. Zwei Geräte im selben Büro kommunizieren lokal, ein Notebook im Homeoffice erreicht einen Cloud-Server direkt. Das senkt die Latenz und entlastet die Internetanbindung des Hauptstandorts.

### Bestehende Netze anbinden

Nicht jedes Gerät kann einen Tailscale-Client ausführen, etwa Drucker, NAS-Systeme älterer Bauart oder Industriesteuerungen. Für solche Fälle übernimmt ein Subnet Router die Rolle eines Gateways: Ein Linux-Server im Zielnetz kündigt das lokale Subnetz im Tailnet an (advertise), und berechtigte Geräte erreichen es darüber.

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
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `curl -fsSL …/install.sh \| sh` | Lädt das offizielle Installationsskript und richtet das Paket-Repository der Distribution ein |
| `net.ipv4.ip_forward = 1` | Erlaubt dem Linux-Kernel, Pakete zwischen Schnittstellen weiterzuleiten; ohne diese Einstellung funktioniert kein Subnet Router |
| `sysctl -p <datei>` | Lädt die Einstellung sofort, ohne Neustart |
| `tailscale up` | Meldet das Gerät am Tailnet an; beim ersten Aufruf erscheint ein Anmeldelink |
| `--advertise-routes=<subnetz>` | Kündigt das angegebene Subnetz im Tailnet an; mehrere Subnetze werden durch Komma getrennt |
| `--advertise-tags=<tag>` | Weist dem Gerät ein Tag zu, auf das sich die Zugriffsregeln beziehen |

</details>

Die angekündigte Route muss anschliessend in der Admin-Konsole freigegeben werden, sofern keine automatische Freigabe (`autoApprovers`) in der Policy konfiguriert ist. Analog lässt sich ein Gerät mit `--advertise-exit-node` als Exit Node betreiben. Clients, die diesen Exit Node wählen, leiten dann ihren gesamten Internetverkehr darüber, was dem Full-Tunnel-Modus eines klassischen VPN entspricht.

### Weniger Betriebsaufwand

Schlüsselrotation, Adressvergabe, DNS und Routing verwaltet der Coordination Server. Ein neuer Standort braucht einen Subnet Router mit Internetzugang, aber keine feste öffentliche IP-Adresse und keine Abstimmung von IPsec-Parametern mit der Gegenseite. Für den Fernzugriff auf Server stehen zusätzlich Tailscale SSH (Anmeldung per Tailnet-Identität ohne verteilte SSH-Schlüssel) und `tailscale serve` (Freigabe eines lokalen Webdienstes im Tailnet) zur Verfügung.

## Grenzen und Abhängigkeiten

Tailscale ersetzt das klassische VPN-Gateway, verlagert aber einen Teil der Verantwortung auf einen externen Dienst. Diese Punkte sollten Sie vor einer Einführung prüfen.

| Thema | Was zu beachten ist |
|---|---|
| Abhängigkeit vom Anbieter | Die Control Plane ist ein Clouddienst von Tailscale Inc. Bei einem Ausfall laufen bestehende Verbindungen weiter, neue Geräte, Anmeldungen und Policy-Änderungen sind bis zur Wiederherstellung nicht möglich |
| Metadaten | Gerätenamen, Tailnet-Adressen, öffentliche IP-Adressen, Benutzerkonten und Verbindungszeitpunkte werden beim Anbieter verarbeitet; die Nutzdaten nicht |
| Vertrauen in die Schlüsselverteilung | Der Coordination Server bestimmt, welche öffentlichen Schlüssel ein Gerät akzeptiert. Tailnet Lock verlangt dafür zusätzlich eine Signatur durch vertrauenswürdige eigene Geräte |
| Quellcode | Der Client-Kern ist quelloffen (BSD-3-Clause), der Coordination Server nicht. Headscale ist eine quelloffene, selbst gehostete Alternative mit reduziertem Funktionsumfang |
| Adresskonflikte | `100.64.0.0/10` wird auch von manchen Providern für Carrier-Grade NAT und von anderen VPN-Produkten verwendet; Überschneidungen führen zu Routing-Problemen |
| Koexistenz mit anderen VPN-Clients | Ein zweiter VPN-Client mit Full Tunnel kann den Tailscale-Verkehr umleiten. Der Bereich `100.64.0.0/10` und der Tailscale-Dienst müssen dort vom Tunnel ausgenommen werden |
| Leistung über Relays | Kommt keine direkte Verbindung zustande, sinkt der Durchsatz über DERP spürbar; `tailscale netcheck` zeigt, ob UDP nach aussen funktioniert |
| Kein Ersatz für Web-Filterung | Tailscale regelt den Zugriff auf interne Ressourcen. Inhaltsfilterung und Inspektion des Internetverkehrs übernimmt es nicht |

Für Unternehmen in der Schweiz gilt das revidierte Datenschutzgesetz (revDSG). Da beim Anbieter Personendaten der Benutzer (Konten, Geräte, Verbindungsmetadaten) anfallen, führen Sie Tailscale im Verzeichnis der Bearbeitungstätigkeiten auf, sofern Ihr Unternehmen eines führen muss (ab 250 Mitarbeitenden oder bei Bearbeitungen mit hohem Risiko). Prüfen Sie den Auftragsbearbeitungsvertrag (Data Processing Addendum) des Anbieters und die Rechtsgrundlage der Bekanntgabe ins Ausland gemäss Art. 16 revDSG, etwa eine Zertifizierung unter dem Swiss-U.S. Data Privacy Framework oder Standardvertragsklauseln. Ist eine Datenbearbeitung beim Anbieter ausgeschlossen, bleibt Headscale als selbst betriebene Control Plane.

> **EU-Hinweis:** Für Niederlassungen in der EU gelten die DSGVO und deren Regeln zur Übermittlung in Drittländer (Art. 44 ff. DSGVO). Die Prüfung verläuft inhaltlich ähnlich; massgebend ist dort das EU-U.S. Data Privacy Framework.

## Kosten

Der Tarif Personal ist kostenlos und umfasst bis zu sechs Benutzer, beliebig viele Benutzergeräte und 50 getaggte Geräte (Stand Oktober 2026). Für den geschäftlichen Einsatz kostet der Tarif Standard 8 USD, Premium 18 USD pro Benutzer und Monat. Premium ergänzt unter anderem Network Flow Logs, Log-Streaming und erweiterte Optionen für Tailscale SSH. Enterprise wird individuell angeboten.

## Für wen sich Tailscale eignet

Tailscale eignet sich besonders, wenn Benutzer und Ressourcen verteilt sind: Homeoffice, Cloud-Server bei mehreren Anbietern, kleine Aussenstandorte ohne feste IP-Adresse oder Fernwartung bei Kunden. Für kleine und mittlere Unternehmen kann es ein VPN-Gateway vollständig ersetzen. In grösseren Umgebungen läuft es häufig parallel zum bestehenden VPN, zum Beispiel für den Administrationszugriff auf Server, wo die feingranularen Zugriffsregeln den grössten Nutzen bringen.

Weniger geeignet ist es, wenn Vorgaben eine vollständig selbst betriebene Infrastruktur verlangen und Headscale den Funktionsumfang nicht abdeckt, oder wenn der gesamte Internetverkehr zentral gefiltert werden muss. In diesem Fall bleibt ein Secure Web Gateway nötig, das Tailscale ergänzt.

Zum Testen genügt es, den Client auf zwei Geräten zu installieren und sich mit demselben Konto anzumelden. Mit `tailscale status` sehen Sie anschliessend alle Geräte im Tailnet, mit `tailscale ping <gerät>` prüfen Sie, ob eine direkte Verbindung oder ein Relay verwendet wird.

## Quellen

1.  [Tailscale: How Tailscale works](https://tailscale.com/blog/how-tailscale-works): Architektur mit Coordination Server, WireGuard-Mesh und lokaler Durchsetzung der Regeln.

2.  [Tailscale: How NAT traversal works](https://tailscale.com/blog/how-nat-traversal-works): ausführliche Erklärung von STUN, UDP Hole Punching und den Fällen, in denen ein Relay nötig wird.

3.  [Tailscale Docs: DERP servers](https://tailscale.com/kb/1232/derp-servers): Funktion und Verschlüsselung der Relay-Server.

4.  [Tailscale Docs: Tailscale Peer Relays](https://tailscale.com/kb/1591/peer-relays): eigene Geräte als Relay mit Vorrang vor DERP.

5.  [Tailscale Docs: Grants](https://tailscale.com/kb/1324/grants): Syntax der Zugriffsregeln in der Policy-Datei.

6.  [Tailscale Docs: Subnet routers](https://tailscale.com/kb/1019/subnets): Einrichtung von Subnet Routern inklusive IP-Forwarding und Freigabe der Routen.

7.  [Tailscale Docs: Exit nodes](https://tailscale.com/kb/1103/exit-nodes): gesamten Internetverkehr über ein Gerät im Tailnet leiten.

8.  [Tailscale Docs: Tailnet Lock](https://tailscale.com/kb/1226/tailnet-lock): Signatur neuer Geräte durch eigene, vertrauenswürdige Nodes.

9.  [Tailscale: Pricing](https://tailscale.com/pricing): Tarife und Limits, abgerufen am 1. Oktober 2026.

10.  [WireGuard: Next Generation Kernel Network Tunnel (Whitepaper)](https://www.wireguard.com/papers/wireguard.pdf): Protokolldesign und verwendete kryptografische Verfahren.

11.  [GitHub: tailscale/tailscale](https://github.com/tailscale/tailscale): Quellcode des Clients unter BSD-3-Clause.

12.  [GitHub: juanfont/headscale](https://github.com/juanfont/headscale): quelloffene Implementierung des Coordination Servers zum Selbstbetrieb.

13.  [CISA: Emergency Directive 24-01](https://www.cisa.gov/news-events/directives/ed-24-01-mitigate-ivanti-connect-secure-and-ivanti-policy-secure-vulnerabilities): Anordnung zur Trennung verwundbarer Ivanti-VPN-Gateways im Januar 2024.

14.  [Fedlex: Bundesgesetz über den Datenschutz (DSG)](https://www.fedlex.admin.ch/eli/cc/2022/491/de): Art. 16 zur Bekanntgabe von Personendaten ins Ausland.
