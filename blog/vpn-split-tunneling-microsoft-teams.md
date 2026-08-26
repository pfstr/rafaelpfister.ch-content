---
title: "VPN-Split-Tunneling für Microsoft Teams: Medienverkehr am Tunnel vorbeiführen"
navTitle: "Teams Split Tunneling"
description: "Teams-Anrufe über ein VPN leiden unter Latenz, Jitter und dem Umweg über das VPN-Gateway. Der Artikel zeigt, welche Microsoft-Netze und Ports für den Medienverkehr zuständig sind, warum IP-basiertes Split Tunneling dem App-Ausschluss überlegen ist und wie die Umsetzung in Consumer-VPNs, WireGuard, OpenVPN und Enterprise-Clients aussieht."
date: "2026-08-26"
kategorie: "Microsoft Teams"
timeToRead: "8 Min. Lesezeit"
themen:
  - "microsoft-teams"
  - "microsoft-365-exchange"
produkte:
  - "teams"
protokolle:
  - "tcp"
hauptthema: "microsoft-teams"
slug: "vpn-split-tunneling-microsoft-teams"
translationId: "article-d15f1e7ff6af231c"
url: "https://rafaelpfister.ch/blog/vpn-split-tunneling-microsoft-teams"
aiPrompt: |
  Du bist mein Netzwerk-Assistent. Ich will Microsoft-Teams-Medienverkehr per Split Tunneling an meinem VPN vorbeiführen. Hilf mir Schritt für Schritt: 1. Frage mich, welchen VPN-Client ich einsetze (Consumer-VPN, WireGuard, OpenVPN, Enterprise-Client) und auf welchem Betriebssystem. 2. Nenne mir die passende Konfiguration für die drei Optimize-Netze 13.107.64.0/18, 52.112.0.0/14 und 52.122.0.0/15 (UDP 3478 bis 3481, TCP 443). 3. Erkläre mir, wie ich mit Find-NetRoute oder der Anrufintegrität in Teams prüfe, ob der Medienverkehr tatsächlich am Tunnel vorbeiläuft. 4. Weise mich auf die Sicherheitsabwägungen hin, bevor ich die Ausnahme produktiv setze.
---

Ein Teams-Anruf über eine VPN-Verbindung klingt oft schlechter als ohne: Die Stimme setzt aus, Video ruckelt, Bildschirmübertragungen bauen sich verzögert auf. Die Ursache liegt meist am Umweg, den der Echtzeitverkehr durch den VPN-Tunnel nimmt, nicht bei Teams selbst. Microsoft empfiehlt darum seit Jahren, den Teams-Medienverkehr per Split Tunneling am VPN vorbei direkt ins Internet zu führen. Dieser Ansatz funktioniert mit praktisch jedem VPN-Produkt, vom Consumer-Client bis zum Enterprise-Gateway; die Konfiguration unterscheidet sich nur im Detail.

## Warum Echtzeitverkehr im Tunnel leidet

Teams-Audio und -Video laufen über SRTP, ein UDP-basiertes Protokoll, das auf niedrige Latenz und geringen Jitter angewiesen ist. Microsoft nennt als Zielwerte unter 100 ms Round-Trip-Zeit zum nächsten Microsoft-Netzwerkeingang und unter 30 ms Jitter. Ein VPN-Tunnel verschlechtert beide Werte gleich mehrfach.

Erstens verlängert der Tunnel den Weg: Statt direkt zum geografisch nächsten Microsoft-Einstiegspunkt läuft der Verkehr zuerst zum VPN-Gateway, das im Rechenzentrum des Anbieters oder der Firma stehen kann, und erst von dort zu Microsoft. Zweitens kostet die zusätzliche Verschlüsselungsschicht Rechenzeit und erhöht den Overhead pro Paket; der Medienstrom ist mit SRTP bereits verschlüsselt, die VPN-Verschlüsselung kommt als zweite Schicht hinzu. Drittens ist das VPN-Gateway ein gemeinsam genutzter Engpass: In Stosszeiten teilen sich alle Benutzer dessen Bandbreite und Paketpuffer, was genau den Jitter erzeugt, auf den Echtzeitverkehr am empfindlichsten reagiert. Viertens blockieren manche VPN-Konfigurationen UDP ganz oder erzwingen TCP; Teams weicht dann auf TCP 443 aus, was die Qualität nochmals verschlechtert, weil TCP-Neuübertragungen für Echtzeitmedien ungeeignet sind.

Für den übrigen Teams-Verkehr (Anmeldung, Chat, Dateizugriff) spielt das alles kaum eine Rolle, weil er nicht echtzeit-sensitiv ist. Es genügt darum, gezielt den Medienverkehr auszunehmen.

## Die relevanten Netze und Ports

Microsoft publiziert alle Microsoft-365-Endpunkte maschinenlesbar und teilt sie in die Kategorien Optimize, Allow und Default ein. Für Split Tunneling relevant ist die Kategorie Optimize: Sie umfasst die wenigen, latenzkritischen Endpunkte mit festen IP-Netzen, die zusammen den grössten Teil des Volumens ausmachen. Für Teams-Medien sind das die Endpunkt-IDs 11 und 12 der offiziellen Liste:

| Netz | Protokoll und Ports | Zweck |
|---|---|---|
| `13.107.64.0/18` | UDP 3478 bis 3481, TCP 443 | Teams-Medien (Audio, Video, Bildschirmübertragung) |
| `52.112.0.0/14` | UDP 3478 bis 3481, TCP 443 | Teams-Medien und Transport-Relays |
| `52.122.0.0/15` | UDP 3478 bis 3481, TCP 443 | Teams-Medien und Transport-Relays |
| `2603:1063::/38` | UDP 3478 bis 3481, TCP 443 | dieselben Dienste über IPv6 |

Die vier UDP-Ports stehen für die Medienklassen Audio (3478), Video (3479 und 3480) und Bildschirmübertragung (3481); TCP 443 ist der Rückfallweg. Wer IPv6 im Einsatz hat, sollte das IPv6-Netz mit ausnehmen, sonst läuft ein Teil der Verbindungen doch wieder durch den Tunnel.

Diese Netze sind bewusst stabil: Microsoft kündigt Änderungen an den Optimize-Endpunkten über den Endpoint-Webservice an und hält die Liste klein, gerade damit Firmen sie in Routing- und Firewall-Regeln giessen können. Trotzdem gehört ein gelegentlicher Abgleich mit der offiziellen Liste in die Betriebsroutine.

## App-basiert oder IP-basiert: zwei Ansätze mit ungleichen Stärken

Viele VPN-Clients bieten zwei Arten von Split Tunneling an: Ausnahmen pro Anwendung oder Ausnahmen pro Ziel-IP.

Der App-Ausschluss klingt naheliegend, hat bei Teams aber zwei Schwächen. Das neue Teams ist eine WebView2-Anwendung: Der Hauptprozess heisst `ms-teams.exe`, ein Teil des Verkehrs läuft aber über `msedgewebview2.exe`. Wer nur den Hauptprozess ausnimmt, erwischt nicht den ganzen Verkehr; wer WebView2 mit ausnimmt, leitet auch den Verkehr anderer WebView2-Anwendungen (etwa das neue Outlook) am Tunnel vorbei. Und für Teams im Browser hilft der App-Ausschluss gar nicht, ausser man nimmt den ganzen Browser aus, womit sämtlicher Web-Verkehr das VPN umgeht.

Der IP-basierte Ausschluss greift dagegen auf Netzwerkebene und ist damit unabhängig davon, ob der Verkehr aus der Teams-App, aus WebView2 oder aus einem Browser-Tab stammt. Er nimmt genau das aus, was latenzkritisch ist, und lässt Anmeldung, Chat und den restlichen Web-Verkehr im Tunnel. Für Teams ist der IP-basierte Ansatz darum die bessere Wahl; der App-Ausschluss taugt als Ergänzung, wenn wirklich der gesamte Teams-Verkehr das VPN umgehen soll.

## Umsetzung in gängigen VPN-Produkten

Das Prinzip ist überall gleich: Die drei IPv4-Netze (und bei Bedarf das IPv6-Netz) werden vom Tunnel ausgenommen, sodass die Betriebssystem-Routen für diese Ziele auf das physische Interface zeigen.

**Consumer-VPNs (Proton VPN, NordVPN, Surfshark und ähnliche):** Die Windows- und Android-Clients bieten meist einen Menüpunkt wie „Split Tunneling" mit einer Ausschlussliste für IP-Adressen oder Subnetze. Dort die drei Netze in CIDR-Notation eintragen und die VPN-Verbindung neu aufbauen, damit die Routen greifen. Auf macOS und iOS fehlt die Funktion bei den meisten Anbietern, weil die System-APIs dort kein anwendungsgesteuertes Split Tunneling in dieser Form zulassen.

**WireGuard:** WireGuard kennt keine Ausschlussliste, sondern nur die `AllowedIPs`-Angabe, die festlegt, was in den Tunnel geht. Ausnahmen entstehen, indem `0.0.0.0/0` durch die Liste aller Netze ersetzt wird, die den Ausschluss-Bereich nicht enthalten. Diese Komplementärliste rechnet niemand von Hand; Online-Rechner wie der WireGuard AllowedIPs Calculator nehmen `0.0.0.0/0` als Basis, die drei Microsoft-Netze als „Disallowed IPs" und liefern die fertige Zeile für die Konfigurationsdatei.

**OpenVPN:** Bei aktivem `redirect-gateway` gewinnen spezifischere Routen. Drei zusätzliche Zeilen in der Client-Konfiguration leiten die Microsoft-Netze am Tunnel vorbei:

```text
route 13.107.64.0 255.255.192.0 net_gateway
route 52.112.0.0 255.252.0.0 net_gateway
route 52.122.0.0 255.254.0.0 net_gateway
```

`net_gateway` steht dabei für das Standard-Gateway des lokalen Netzes, nicht für das VPN-Gateway.

**Enterprise-Clients (Cisco Secure Client/AnyConnect, Palo Alto GlobalProtect, Fortinet FortiClient):** Hier konfiguriert die Firma die Ausnahmen zentral, bei Cisco als „Split Exclude"-Liste in der Gruppenrichtlinie, bei GlobalProtect als „Exclude Access Route". Microsoft dokumentiert dieses Vorgehen ausdrücklich als empfohlenes Modell für Microsoft-365-Verkehr und liefert die Optimize-Liste über den Endpoint-Webservice, sodass sich die Ausnahmen automatisiert aktuell halten lassen. Wer als Mitarbeiter hinter einem Firmen-VPN sitzt, kann die Ausnahme also nicht selbst setzen, sondern muss sie beim Netzwerk-Team beantragen; das Microsoft-Dokument dazu ist die passende Argumentationsgrundlage.

**Windows-Bordmittel:** Bei einer mit Windows-Bordmitteln eingerichteten VPN-Verbindung im Split-Modus (`Set-VpnConnection -SplitTunneling $true`) landen nur die per `Add-VpnConnectionRoute` eingetragenen Netze im Tunnel. Solange die Microsoft-Netze dort nicht auftauchen, laufen sie automatisch direkt; ein expliziter Ausschluss ist dann unnötig.

## Sicherheitsabwägung: was am Tunnel vorbeigeht

Split Tunneling ist eine bewusste Aufweichung des Grundsatzes, allen Verkehr durch den Tunnel zu führen. Vor der Umsetzung sollten Sie drei Punkte klären.

Die eigene öffentliche IP-Adresse wird für Microsoft sichtbar, denn genau das ist beabsichtigt: Der Medienstrom soll den kürzesten Weg nehmen. Wer ein VPN primär einsetzt, um den eigenen Standort zu verbergen, gibt diesen Schutz für Teams-Anrufe auf. Die Inhalte bleiben davon unberührt, weil SRTP den Medienstrom Ende-zu-Ende zwischen Client und Microsoft-Infrastruktur verschlüsselt.

Im Firmenumfeld verliert das zentrale Security-Gateway die Sicht auf den ausgenommenen Verkehr: TLS-Inspektion, IDS-Signaturen und Volumen-Auswertung greifen für diese Netze nicht mehr. Da die Ausnahme auf wenige, fest Microsoft zugeordnete Netze mit definierten Ports beschränkt ist, bewertet Microsoft dieses Restrisiko als gering; die Optimize-Endpunkte sind genau dafür kuratiert. Eine pauschale Ausnahme ganzer Anwendungen oder gar des Browsers hat dagegen eine deutlich grössere Angriffsfläche und sollte im Firmenumfeld unterbleiben.

Und schliesslich der Kill Switch: Manche VPN-Clients setzen Split-Tunneling-Ausnahmen erst nach einem Neuaufbau der Verbindung um oder verhalten sich mit aktivem Kill Switch abweichend. Nach jeder Änderung an der Ausschlussliste gehört darum ein Verbindungsneuaufbau und ein Kontrolltest dazu.

## Kontrolle: läuft der Medienverkehr wirklich direkt?

Ob die Ausnahme greift, lässt sich auf zwei Ebenen prüfen. Auf Routing-Ebene zeigt PowerShell, welches Interface Windows für ein Ziel in den Microsoft-Netzen wählt:

```powershell
Find-NetRoute -RemoteIPAddress 52.112.1.1 |
  Select-Object InterfaceAlias, NextHop
```

Erscheint hier das physische Interface (Ethernet oder WLAN) statt des VPN-Adapters, stimmt die Route. Auf Anwendungsebene liefert Teams selbst die Bestätigung: Während eines Anrufs zeigt die Anrufintegrität (unter „Weitere Aktionen" im Anruffenster) die ausgehandelte Verbindungsart, die Round-Trip-Zeit und die Paketverlustrate. Eine Round-Trip-Zeit, die nach der Umstellung deutlich sinkt, und der Verbindungstyp UDP statt TCP sind die beiden Kennzeichen einer funktionierenden Ausnahme.

Bleibt der Verkehr trotz korrekter Route im Tunnel, lohnt ein Blick auf die Reihenfolge der Netzwerkadapter und auf Client-Eigenheiten: Einige VPN-Clients erzwingen ihre Routen mit niedrigerer Metrik nach jedem Verbindungsaufbau neu, und eine veraltete Ausschlussliste fällt erst auf, wenn Microsoft ein Netz ergänzt. Der Abgleich mit der offiziellen Endpunktliste gehört darum in denselben Rhythmus wie andere wiederkehrende Netzwerk-Prüfungen.

## Quellen

1.  [Microsoft: Office 365 URLs and IP address ranges](https://learn.microsoft.com/en-us/microsoft-365/enterprise/urls-and-ip-address-ranges): offizielle Endpunktliste; die Teams-Medien-Netze stehen unter den IDs 11 und 12 in der Kategorie Optimize.

2.  [Microsoft: Implementing VPN split tunneling for Microsoft 365](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-vpn-implement-split-tunnel): Microsofts Umsetzungsleitfaden für Enterprise-VPNs inklusive Begründung der Risikobewertung.

3.  [Microsoft: Microsoft 365 network connectivity principles](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-network-connectivity-principles): die Grundsätze hinter dem lokalen Internet-Ausstieg, inklusive der Latenz-Zielwerte für Echtzeitmedien.

4.  [Proton VPN: How to use split tunneling](https://protonvpn.com/support/protonvpn-split-tunneling/): Beispiel eines Consumer-Clients mit IP- und App-basiertem Split Tunneling unter Windows und Android.

5.  [WireGuard AllowedIPs Calculator](https://www.procustodibus.com/blog/2021/03/wireguard-allowedips-calculator/): Rechner für die Komplementärliste, wenn Ausnahmen über AllowedIPs abgebildet werden müssen.
