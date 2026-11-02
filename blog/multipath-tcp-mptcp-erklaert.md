---
title: "Multipath TCP (MPTCP) unter Linux: Funktionsweise, Geschichte und Prüfung vor dem Abschalten"
navTitle: "MPTCP unter Linux"
description: "Multipath TCP verteilt eine TCP-Verbindung auf mehrere Netzwerkpfade. Seit Go 1.24 lauschen viele Go-Dienste unter Linux per MPTCP, ohne dass jemand das eingerichtet hat. Mit CVE-2026-98432 ist das relevant geworden. Der Artikel erklärt das Protokoll, seine Geschichte und wie Sie prüfen, ob Sie MPTCP abschalten können."
date: "2026-11-02"
kategorie: "Linux-Kernel"
timeToRead: "10 Min. Lesezeit"
themen:
  - "linux-kernel"
  - "sicherheitsluecken"
produkte:
  - "linux-kernel"
  - "sicherheitsluecken"
protokolle:
  - "tcp"
  - "haertung"
hauptthema: "linux-kernel"
slug: "multipath-tcp-mptcp-erklaert"
translationId: "article-1caab37cb0d7fce7"
url: "https://rafaelpfister.ch/blog/multipath-tcp-mptcp-erklaert"
---

Beim Prüfen meines Heim-Servers auf CVE-2026-98432 habe ich festgestellt, dass dort mehrere Dienste per Multipath TCP lauschten, darunter der Prometheus-node-exporter, der Pi-hole-Exporter und `tailscaled`, alle in Go geschrieben. Eingerichtet hatte ich das nie. Der bisher öffentlich beschriebene Root-Exploit für diese Lücke setzt genau einen solchen MPTCP-Listener voraus.

Kurzfazit: MPTCP ist unter Linux seit Kernel 5.6 im offiziellen Kernel und standardmässig erlaubt. Seit Go 1.24 öffnen Go-Programme ihre Listener automatisch als MPTCP-Sockets. Ob Sie MPTCP abschalten können, lässt sich mit zwei Befehlen und einigen Zählern des Kernels klären. Auf den meisten Servern nutzt kein Client das Protokoll, und Go fällt nach dem Abschalten von selbst auf normales TCP zurück.

## Was Multipath TCP ist

Eine normale TCP-Verbindung ist an genau ein Adresspaar gebunden: Quell-IP und Port, Ziel-IP und Port. Wechselt ein Smartphone vom WLAN ins Mobilnetz, ändert sich die Quell-IP, und die Verbindung bricht ab. Multipath TCP erweitert TCP so, dass eine Verbindung mehrere Pfade gleichzeitig oder nacheinander nutzen kann, etwa WLAN und Mobilfunk.

Technisch besteht eine MPTCP-Verbindung aus mehreren `Subflows`. Jeder Subflow ist für das Netz eine gewöhnliche TCP-Verbindung über einen Pfad. Die Anwendung sieht davon nichts: Sie arbeitet mit einem einzigen Socket, und der Kernel verteilt die Daten auf die Subflows. Damit die Daten in der richtigen Reihenfolge ankommen, führt MPTCP eine eigene 64-Bit-Sequenznummer auf Verbindungsebene (Data Sequence Number) zusätzlich zu den Sequenznummern der einzelnen Subflows.

Ausgehandelt wird MPTCP im normalen TCP-Handshake. SYN, SYN/ACK und das erste ACK tragen die TCP-Option `MP_CAPABLE` (Option Kind 30), mit der beide Seiten erklären, dass sie MPTCP sprechen, und Schlüssel austauschen. Weitere Subflows treten der Verbindung später mit der Option `MP_JOIN` bei. Fehlt `MP_CAPABLE` in der Antwort, weil die Gegenseite MPTCP nicht kennt oder eine Middlebox die Option entfernt, läuft die Verbindung als gewöhnliches TCP weiter. Dieser Fallback ist Teil des Standards und der Grund, weshalb MPTCP-fähige Server mit allen TCP-Clients funktionieren.

Unter Linux ist MPTCP Opt-in pro Socket: Eine Anwendung muss den Socket ausdrücklich mit dem Protokoll `IPPROTO_MPTCP` (Wert 262) anlegen. Ein Programm, das das nicht tut, verwendet weiterhin normales TCP, auch wenn MPTCP im Kernel aktiv ist.

## Die Geschichte

MPTCP entstand an Universitäten und kam über Apple in grossem Massstab in Betrieb. Die Linux-Implementierung lag über ein Jahrzehnt ausserhalb des offiziellen Kernels, bevor sie in neuer Form übernommen wurde.

| Zeitpunkt | Ereignis |
|---|---|
| ab etwa 2009 | Forschende der Université catholique de Louvain (UCLouvain) entwickeln die erste Linux-Implementierung als eigenen Kernel-Zweig (multipath-tcp.org). |
| März 2011 | RFC 6182 beschreibt die Architektur von Multipath TCP (Informational). |
| Januar 2013 | RFC 6824 spezifiziert MPTCP Version 0 als experimentelles Protokoll. |
| September 2013 | iOS 7 erscheint mit MPTCP, der erste kommerzielle Einsatz in grossem Umfang. Apple nutzt MPTCP unter anderem für Siri. |
| Juni 2015 | KT (Korea Telecom) startet einen kommerziellen Dienst, der WLAN und LTE per MPTCP bündelt und Smartphones bis 1 Gbit/s ermöglicht; vorgestellt an der IETF 93. |
| 2017 | iOS 11 öffnet MPTCP für App-Entwickler über `URLSession` (`multipathServiceType`). |
| März 2020 | RFC 8684 definiert MPTCP Version 1 als Proposed Standard und ersetzt RFC 6824. |
| 29. März 2020 | Linux 5.6 erscheint mit einer ersten MPTCP-Unterstützung im offiziellen Kernel, neu geschrieben und nur für Version 1. Zusätzliche Subflows kamen in den folgenden Versionen dazu. |
| 2020 | 3GPP spezifiziert in Release 16 ATSSS (Access Traffic Steering, Switching, Splitting) für 5G; MPTCP verteilt dort Verkehr auf Mobilfunk und WLAN. |
| August 2023 | Go 1.21 führt `SetMultipathTCP` für Dialer und Listener ein, standardmässig aus. |
| Februar 2025 | Go 1.24 schaltet MPTCP für Listener standardmässig ein (`ListenConfig`), Dialer bleiben bei TCP. |
| Oktober 2026 | CVE-2026-98432: Use-after-free im IPv4-TCP-Code; der gezeigte Root-Exploit braucht einen MPTCP-Listener. |

Der Out-of-Tree-Zweig der UCLouvain unterstützte Version 0 und 1. Der offizielle Kernel kennt nur Version 1; das Projekt empfiehlt heute ausschliesslich diese Implementierung.

## Warum MPTCP auf vielen Servern aktiv ist

Zwei Voreinstellungen treffen zusammen. Der Kernel erlaubt MPTCP-Sockets standardmässig: Die Sysctl-Einstellung `net.mptcp.enabled` steht auf `1`. Und Go-Programme ab Version 1.24 legen jeden Listener als MPTCP-Socket an, sofern der Kernel das zulässt. Go-Dienste wie Prometheus-Exporter oder Tailscale lauschen deshalb per MPTCP, sobald sie mit Go 1.24 oder neuer gebaut wurden.

Für normale Clients ändert das nichts: Sie senden kein `MP_CAPABLE`, und die Verbindung läuft als TCP. Die zusätzliche Angriffsfläche entsteht jedoch auf der Serverseite, weil jeder Client, der `MP_CAPABLE` mitschickt, den MPTCP-Code im Kernel erreicht.

## CVE-2026-98432

CVE-2026-98432 ist ein Use-after-free beim Aufbau von IPv4-TCP-Verbindungen mit IP-Optionen (CVSS 3.1: 9.8). Betroffen ist laut CVE-Eintrag Linux ab Version 4.4. Der Fehler selbst betrifft jeden TCP-Listener, nicht nur MPTCP. Für die Ausnutzung bis zu Root-Rechten wurde jedoch bisher nur eine Kette über einen MPTCP-Listener gezeigt; laut lilting.ch hat Anthropic sie am 27. Oktober 2026 in einer Laborumgebung vorgeführt. Ob Root-Rechte auch ohne MPTCP erreichbar sind, ist offen.

Upstream ist der Fehler in den Stable-Kerneln 5.10.272, 5.15.223, 6.1.190, 6.6.159, 6.12.113, 6.18.56 und 7.2.10 sowie in Mainline 7.3 behoben (Commit „ipv4: free inet_opt and ireq_opt after an RCU grace period"). Ubuntu führte am 30. Oktober 2026 die LTS-Versionen 22.04, 24.04 und 26.04 noch als „Vulnerable, work in progress"; für 26.04 ist der Fix in `linux` 7.0.0-43.43 vorgesehen. Distributionen portieren Fixes zurück; massgebend ist deshalb die Paketversion aus dem Advisory der Distribution, nicht die Kernel-Hauptversion.

MPTCP abzuschalten entfernt die Voraussetzung des gezeigten Exploits. Die Lücke im IPv4-TCP-Code bleibt bestehen, bis der aktualisierte Kernel installiert und gebootet ist.

## Prüfen, ob MPTCP aktiv ist und genutzt wird

### 1. Lässt der Kernel MPTCP zu?

```bash
sysctl net.mptcp.enabled
```

`1` heisst, Anwendungen dürfen MPTCP-Sockets anlegen. Meldet `sysctl`, der Schlüssel sei unbekannt, ist MPTCP nicht in den Kernel kompiliert und auf diesem Host nicht verfügbar.

### 2. Welche Dienste lauschen per MPTCP?

```bash
sudo ss -Mlnp
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-M` | nur MPTCP-Sockets anzeigen |
| `-l` | nur lauschende Sockets (Listener) |
| `-n` | Adressen und Ports numerisch, ohne Namensauflösung |
| `-p` | Prozess zum Socket anzeigen; braucht Root-Rechte für Prozesse anderer Benutzer |

</details>

Jede Zeile ist ein MPTCP-Listener. Derselbe Port erscheint mit `ss -tln` zusätzlich als TCP-Listener; das ist normal, weil der Kernel für jeden MPTCP-Listener intern einen TCP-Socket führt. Die Prozessnamen zeigen, welche Programme betroffen sind. In der Regel sind es Go-Programme.

### 3. Hat je ein Client MPTCP genutzt?

Der Kernel zählt MPTCP-Handshakes seit dem letzten Start. Die relevanten Zähler:

```bash
nstat -asz |
  grep -E 'MPTcpExt(MPCapable|MPCurrEstab)'
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-a` | absolute Werte seit dem Systemstart statt Differenz zum letzten Aufruf |
| `-s` | Verlaufsdatei von `nstat` nicht aktualisieren |
| `-z` | auch Zähler mit dem Wert 0 ausgeben |

</details>

| Zähler | Bedeutung |
|---|---|
| `MPTcpExtMPCapableSYNRX` | eingehende SYN mit `MP_CAPABLE` an einem MPTCP-Listener |
| `MPTcpExtMPCapableACKRX` | eingehende MPTCP-Verbindungen, deren Handshake abgeschlossen wurde |
| `MPTcpExtMPCapableSYNTX` | ausgehende SYN mit `MP_CAPABLE`, also Verbindungen, die dieser Host per MPTCP aufbauen wollte |
| `MPTcpExtMPCapableSYNACKRX` | ausgehende Verbindungen, bei denen die Gegenseite MPTCP bestätigt hat |
| `MPTcpExtMPCapableFallbackACK` | serverseitig während des Handshakes auf TCP zurückgefallene Verbindungen |
| `MPTcpExtMPCapableFallbackSYNACK` | clientseitig zurückgefallene Verbindungen |
| `MPTcpExtMPCurrEstab` | aktuell bestehende MPTCP-Verbindungen |

Stehen `MPCapableSYNRX` und `MPCapableACKRX` nach einer repräsentativen Laufzeit auf 0, hat kein Client MPTCP verwendet. Ist `MPCapableSYNTX` grösser als 0, baut eine Anwendung auf dem Host selbst ausgehende MPTCP-Verbindungen auf; diese Anwendung sollten Sie vor dem Abschalten identifizieren. Bestehende MPTCP-Verbindungen zeigt `ss -Mani` mit Details zu den Subflows.

Die Zähler gelten seit dem letzten Neustart. Ein Server, der erst vor einer Stunde gebootet wurde, liefert deshalb keine belastbare Aussage. Zudem zählt der Kernel pro Netzwerk-Namespace, mehr dazu im nächsten Abschnitt.

### 4. Container separat prüfen

`net.mptcp.enabled` gilt pro Netzwerk-Namespace. Jeder Container mit eigenem Netzwerk (Docker-Bridge, Podman, Kubernetes-Pods) hat einen eigenen Wert, und ein neuer Namespace beginnt mit dem Standardwert `1`, unabhängig von der Einstellung auf dem Host. Auf meinem Heim-Server standen nach dem Abschalten auf dem Host alle Bridge-Container weiterhin auf `1`; nur die Container mit `network_mode: host` übernahmen die `0`.

```bash
for c in $(docker ps -q); do
  pid=$(docker inspect -f '{{.State.Pid}}' "$c")
  name=$(docker inspect -f '{{.Name}}' "$c")
  echo "$name $(sudo nsenter -t "$pid" -n sysctl -n net.mptcp.enabled)"
done
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `docker ps -q` | IDs aller laufenden Container |
| `docker inspect -f '{{.State.Pid}}'` | PID des Hauptprozesses im Container, über die `nsenter` den Namespace findet |
| `nsenter -t <pid>` | Namespaces des angegebenen Prozesses betreten |
| `-n` | nur den Netzwerk-Namespace betreten; Dateisystem und Werkzeuge bleiben die des Hosts |
| `sysctl -n` | nur den Wert ausgeben, ohne Schlüsselnamen |

</details>

`nsenter` hat den Vorteil, dass im Container weder `sysctl` noch `cat` vorhanden sein muss; viele Images (distroless, scratch) enthalten keine Shell. Auf dieselbe Weise lassen sich `ss -Mln` und die `nstat`-Zähler im Namespace eines Containers abfragen.

## Wann Sie MPTCP abschalten können

Das Abschalten ist unproblematisch, wenn die Zähler aus Schritt 3 auf dem Host und in den Containern keine MPTCP-Nutzung zeigen. Lauschende Go-Programme verlieren dadurch nichts: Go versucht den MPTCP-Socket, erhält vom Kernel einen Fehler und legt still einen TCP-Socket an. Andere Programme, die `IPPROTO_MPTCP` ausdrücklich anfordern, erhalten den Fehler `ENOPROTOOPT`. Ob sie dann auf TCP zurückfallen oder abbrechen, hängt von der Software ab; testen Sie solche Dienste nach der Umstellung.

Vorsicht ist in diesen Fällen angebracht:

- **Hybrid Access und Link-Bündelung:** Router oder Proxys, die DSL, Glasfaser und Mobilfunk per MPTCP bündeln (etwa OpenMPTCProuter oder Hybrid-Access-Gateways von Providern), sind auf MPTCP angewiesen.
- **Server für mobile Apps:** Eine eigene iOS-App mit `multipathServiceType` profitiert beim Netzwechsel nur, wenn der Server MPTCP spricht. Die Zähler `MPCapableACKRX` zeigen, ob solche Clients vorkommen.
- **Ausgehendes MPTCP:** Programme, die mit `mptcpize` gestartet werden oder in Go mit `GODEBUG=multipathtcp=1` oder `3` laufen, bauen ausgehend MPTCP auf. Erkennbar an `MPCapableSYNTX` grösser als 0.

## MPTCP abschalten

Es gibt drei Möglichkeiten, je nach gewünschter Reichweite.

### Für den ganzen Host

```bash
echo "net.mptcp.enabled = 0" |
  sudo tee /etc/sysctl.d/99-no-mptcp.conf
sudo sysctl --system
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `/etc/sysctl.d/99-no-mptcp.conf` | Konfigurationsdatei, die beim Booten gelesen wird; die Einstellung bleibt über Neustarts erhalten |
| `sysctl --system` | alle Dateien unter `/etc/sysctl.d/` und den übrigen Standardpfaden neu laden und sofort anwenden |

</details>

Die Einstellung verhindert nur das Anlegen neuer MPTCP-Sockets. Bereits offene Listener bleiben MPTCP-Sockets, bis der Dienst neu startet. Am einfachsten ist ein Neustart des Hosts; danach muss `ss -Mln` eine leere Liste liefern.

### Für Container

In Containern mit eigenem Netzwerk setzen Sie den Wert pro Container. Docker erlaubt Sysctls aus dem Bereich `net.*`, solange der Container nicht das Host-Netzwerk verwendet:

```yaml
services:
  app:
    sysctls:
      net.mptcp.enabled: 0
```

Beim direkten Aufruf entspricht das `docker run --sysctl net.mptcp.enabled=0 …`. Die Änderung greift beim nächsten Neuerstellen des Containers (`docker compose up -d`).

### Nur für einzelne Go-Programme

Statt MPTCP im Kernel abzuschalten, lässt sich das Go-Standardverhalten pro Dienst zurücksetzen. Die Umgebungsvariable `GODEBUG=multipathtcp=0` schaltet MPTCP für Listener und Dialer ab. Bei einem systemd-Dienst geht das über ein Drop-in:

```bash
sudo systemctl edit node_exporter
```

```ini
[Service]
Environment=GODEBUG=multipathtcp=0
```

Setzt der Dienst bereits eine `GODEBUG`-Variable, werden die Einträge mit Komma getrennt in einer Variable kombiniert. Diese Möglichkeit wirkt nur bei Go-Programmen und nur, wenn der Code `SetMultipathTCP` nicht selbst aufruft; ein expliziter Aufruf im Programm hat Vorrang vor `GODEBUG`. Für eigene Go-Projekte steht dieselbe Einstellung als `godebug multipathtcp=0` in der `go.mod` oder als `//go:debug multipathtcp=0` im Hauptpaket zur Verfügung.

## Kontrolle nach der Umstellung

```bash
sysctl net.mptcp.enabled
ss -Mln
```

Erwartet werden `0` und keine Zeile unter dem Tabellenkopf von `ss`. Die Dienste lauschen danach weiterhin auf denselben Ports, sichtbar mit `ss -tln`. Rückgängig machen lässt sich die Umstellung, indem Sie die Datei unter `/etc/sysctl.d/` löschen und den Host neu starten.

## Quellen

1.  [RFC 8684: TCP Extensions for Multipath Operation with Multiple Addresses](https://datatracker.ietf.org/doc/html/rfc8684): Spezifikation von MPTCP Version 1 (März 2020) mit Subflows, `MP_CAPABLE`, Data Sequence Number und Fallback.

2.  [RFC 6824](https://datatracker.ietf.org/doc/rfc6824/): MPTCP Version 0, experimentell (Januar 2013), durch RFC 8684 ersetzt.

3.  [RFC 6182: Architectural Guidelines for Multipath TCP Development](https://datatracker.ietf.org/doc/rfc6182/): Architektur-Dokument von März 2011.

4.  [Linux Kernel Documentation: Multipath TCP](https://docs.kernel.org/networking/mptcp.html): Opt-in über `IPPROTO_MPTCP`, Fehlercodes beim Anlegen von Sockets, Path Manager.

5.  [Linux Kernel Documentation: MPTCP Sysfs variables](https://docs.kernel.org/networking/mptcp-sysctl.html): Bedeutung und Standardwert von `net.mptcp.enabled` und den übrigen MPTCP-Sysctls.

6.  [mptcp.dev](https://www.mptcp.dev/): Website des Upstream-Projekts mit FAQ, Debugging-Anleitung (`ss -Mani`, `nstat`) und Hinweisen zu macOS und iOS.

7.  [KernelNewbies: Linux 5.6](https://kernelnewbies.org/Linux_5.6): Release-Übersicht mit der ersten MPTCP-Unterstützung im offiziellen Kernel.

8.  [Go 1.21 Release Notes](https://go.dev/doc/go1.21): Einführung der MPTCP-Unterstützung im Paket `net`, standardmässig aus, mit stillem Fallback auf TCP.

9.  [Go 1.24 Release Notes](https://go.dev/doc/go1.24): MPTCP standardmässig für `ListenConfig` unter Linux.

10.  [Go: GODEBUG-Einstellungen](https://go.dev/doc/godebug): Werte 0 bis 3 von `multipathtcp` sowie `godebug` in `go.mod` und `//go:debug`.

11.  [Wikipedia: Multipath TCP](https://en.wikipedia.org/wiki/Multipath_TCP): Überblick über Implementierungen und Einsätze (UCLouvain, Apple, KT, 3GPP ATSSS).

12.  [Ubuntu Security: CVE-2026-98432](https://ubuntu.com/security/CVE-2026-98432): Status der Ubuntu-Kernel je Release und vorgesehene Fixversionen.

13.  [lilting.ch: CVE-2026-98432 Linux TCP Use-After-Free](https://lilting.ch/en/articles/linux-ipv4-tcp-options-uaf-mptcp-rce): Einordnung der Ausnutzbarkeit, Rolle von MPTCP und Fixversionen der Stable-Kernel.
