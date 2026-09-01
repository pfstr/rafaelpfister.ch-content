---
title: "RustDesk: die quelloffene TeamViewer-Alternative einrichten"
navTitle: "RustDesk einrichten"
description: "RustDesk ist eine quelloffene Fernwartungssoftware unter AGPL, kostenlos und selbst hostbar. Wie Sie den Client unter Windows installieren (auch unbeaufsichtigt per MSI), wie die Verbindung über den öffentlichen Vermittlungsserver, einen eigenen Server oder eine Direktverbindung zustande kommt, welche Funktionen der Support-Alltag braucht und wo die Grenzen der kostenlosen Nutzung liegen."
date: "2026-09-01"
kategorie: "Fernwartung und Support"
timeToRead: "9 Min. Lesezeit"
themen:
  - "fernwartung"
produkte:
  - "rustdesk"
protokolle:
  - "haertung"
slug: "rustdesk-teamviewer-alternative"
translationId: "article-425ae4b8d562ae41"
url: "https://rafaelpfister.ch/blog/rustdesk-teamviewer-alternative"
aiPrompt: |
  Du bist mein IT-Support-Assistent. Hilf mir, RustDesk als quelloffene TeamViewer-Alternative einzurichten: Client installieren, Verbindungsart wählen (öffentlicher Vermittlungsserver, eigener Server oder Direktverbindung über ein privates Netz), unbeaufsichtigten Zugriff absichern und die Grenzen der kostenlosen Nutzung einordnen.
---
# RustDesk: die quelloffene TeamViewer-Alternative einrichten

TeamViewer und AnyDesk decken Fernwartung zuverlässig ab, verlangen für den gewerblichen Einsatz aber eine Lizenz, und die Preise steigen mit der Zahl der betreuten Geräte. RustDesk ist eine Alternative unter der Lizenz AGPL-3.0: quelloffen, kostenlos und ohne Lizenzzwang. Der Client läuft unter Windows, macOS, Linux, Android und iOS sowie im Browser. Geschrieben ist er in Rust, die Oberfläche in Flutter.

Der entscheidende Unterschied zu den kommerziellen Produkten liegt in der Vermittlung: RustDesk trennt den Client von der Serverinfrastruktur. Sie können den kostenlosen öffentlichen Vermittlungsserver nutzen, einen eigenen Server betreiben oder ganz ohne Vermittlungsserver eine Direktverbindung aufbauen. Damit lässt sich RustDesk vom Einzelplatz bis zur selbst gehosteten Support-Plattform betreiben, ohne dass Verbindungsdaten über einen Anbieter laufen müssen.

## Die drei Verbindungsarten

Bevor Sie installieren, sollten Sie die Verbindungsart festlegen, denn davon hängen Konfiguration und offene Ports ab.

| Verbindungsart | Wie sie funktioniert | Wann sinnvoll |
|---|---|---|
| Öffentlicher Vermittlungsserver | Zwei Clients finden sich über die ID (neunstellige Nummer) am RustDesk-Server, die Verbindung läuft direkt oder über einen Relay | Schneller Einstieg, Test, privater Gelegenheitssupport |
| Eigener Server (self-hosted) | Sie betreiben die Server-Komponenten `hbbs` (Vermittlung) und `hbbr` (Relay) selbst, alle Clients tragen deren Adresse ein | Gewerblicher Einsatz, viele Geräte, volle Datenhoheit |
| Direktverbindung (Direct IP Access) | Der Client verbindet sich ohne Vermittlungsserver direkt auf die IP-Adresse der Gegenstelle | Beide Geräte erreichen sich im selben Netz oder über ein VPN |

Der öffentliche Server ist ausdrücklich für Tests und den privaten Gebrauch gedacht. Für den produktiven, gewerblichen Betrieb empfiehlt das Projekt einen eigenen Server, auch weil der öffentliche Dienst gedrosselt ist und keine Verfügbarkeitszusage trägt.

## Installation unter Windows

Den Installer laden Sie von der offiziellen Quelle, den GitHub-Releases des Projekts (`github.com/rustdesk/rustdesk`). Für Windows gibt es eine ausführbare Datei und ein MSI-Paket. Für die interaktive Installation genügt ein Doppelklick. Wollen Sie RustDesk auf mehreren Rechnern oder im Hintergrund ausbringen, nutzen Sie das MSI mit einer stillen Installation:

```powershell
msiexec /i rustdesk-1.4.9-x86_64.msi /qn /norestart
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `/i <paket>` | Installiert das angegebene MSI-Paket |
| `/qn` | Keine Oberfläche, keine Dialoge (still) |
| `/norestart` | Verhindert einen automatischen Neustart nach der Installation |

</details>

Die stille Installation richtet den Dienst `RustDesk` ein, der beim Systemstart läuft und den unbeaufsichtigten Zugriff ermöglicht. Nach der Installation lesen Sie die ID des Geräts über die Kommandozeile aus, ohne die Oberfläche zu öffnen:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --get-id
```

Ein festes Passwort für den unbeaufsichtigten Zugriff setzen Sie ebenfalls über die Kommandozeile. Vergeben Sie ein eigenständiges, ausreichend langes Passwort, nicht das Anmeldekennwort des Benutzers:

```powershell
& 'C:\Program Files\RustDesk\rustdesk.exe' --password "IhrLangesEinmalpasswort"
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `--get-id` | Gibt die neunstellige RustDesk-ID des Geräts aus |
| `--password <wert>` | Setzt das feste Passwort für unbeaufsichtigten Zugriff |
| `--silent-install` | Installiert die ausführbare Fassung (`.exe`) ohne Oberfläche als Dienst |

</details>

## Eigenen Server eintragen

Betreiben Sie einen eigenen Vermittlungsserver, tragen die Clients dessen Adresse und den öffentlichen Schlüssel ein. In der Oberfläche steht das unter den Netzwerkeinstellungen als ID-Server, Relay-Server und Key. Für die Massenverteilung lässt sich die Konfiguration auch als Datei oder über Umgebungsvariablen vorgeben, sodass jeder Client vorkonfiguriert startet.

Ein eigener Server benötigt die beiden Komponenten `hbbs` und `hbbr`, die meist als Docker-Container laufen. Beide erwarten offene Ports, damit sich Clients registrieren und ein Relay nutzen können.

| Port | Protokoll | Komponente und Zweck |
|---|---|---|
| 21114 | TCP | Weboberfläche der Pro-Fassung (nur dort) |
| 21115 | TCP | `hbbs`, Test des NAT-Typs |
| 21116 | TCP und UDP | `hbbs`, Registrierung (UDP) und Verbindungsaufbau (TCP) |
| 21117 | TCP | `hbbr`, Relay-Verkehr |
| 21118, 21119 | TCP | Unterstützung für Web-Clients |

Öffnen Sie nur die Ports, die Ihre Verbindungsart tatsächlich braucht, und beschränken Sie den Zugriff über die Firewall auf die Netze, aus denen Support geleistet wird.

## Direktverbindung ohne Vermittlungsserver

Erreichen sich beide Geräte im selben Netz oder über ein VPN, kommt RustDesk ganz ohne Vermittlungsserver aus. Aktivieren Sie dazu auf dem Zielgerät den Direktzugriff (in der Oberfläche unter Sicherheit als "IP-Direktzugriff aktivieren", intern der Schalter `direct-server`). Der Client hört dann auf dem Standardport 21118 (TCP). Im Verbindungsfenster geben Sie statt der ID die IP-Adresse der Gegenstelle ein.

Begrenzen Sie den Direktzugriff über die Firewall auf das Netz, aus dem Sie zugreifen. Läuft der Zugriff über ein VPN, geben Sie den Port nur für den VPN-Adressbereich frei, nicht für das gesamte Internet.

## Funktionen im Support-Alltag

RustDesk deckt die Funktionen ab, die Fernwartung im Alltag braucht:

- Bildschirmübertragung und Fernsteuerung von Tastatur und Maus, mit Auswahl des Monitors bei mehreren Bildschirmen.
- Dateiübertragung in beide Richtungen über ein zweigeteiltes Fenster.
- Textchat während der Sitzung.
- Unbeaufsichtigter Zugriff über ein festes Passwort, für Geräte ohne anwesenden Benutzer.
- Sitzungsaufzeichnung als Videodatei, auf Wunsch automatisch.
- TCP-Tunnel und Weiterleitung, um einzelne Dienste der Gegenstelle lokal zu erreichen.
- Adressbuch und mehrere gespeicherte Geräte, in der kostenlosen Fassung lokal, in der Pro-Fassung serverseitig geteilt.

Für den betreuten Support ist wichtig: Standardmässig fragt RustDesk auf der Gegenseite nach, ob die Verbindung angenommen wird, und zeigt während der Sitzung an, dass ein Zugriff läuft. Der Mensch am Gerät weiss also Bescheid. Erst ein festes Passwort für den unbeaufsichtigten Zugriff hebt die Rückfrage auf. Setzen Sie unbeaufsichtigten Zugriff nur auf Geräten ein, deren Nutzer wissen, dass die Software installiert ist und wofür sie dient.

## Einschränkungen und Grenzen

RustDesk ersetzt TeamViewer in vielen Fällen, hat aber Grenzen, die Sie vor dem Einsatz kennen sollten:

- Der öffentliche Vermittlungsserver ist gedrosselt, ohne Verfügbarkeitszusage und für den gewerblichen Dauerbetrieb nicht vorgesehen. Wer verlässlich arbeiten will, hostet selbst.
- Ein eigener Server bedeutet Betriebsaufwand: Container, offene Ports, Zertifikate und Updates liegen bei Ihnen.
- Ein serverseitig geteiltes Adressbuch, zentrale Benutzerverwaltung und die Weboberfläche zur Verwaltung gehören zur Pro-Fassung, die ab einer bestimmten Gerätezahl kostenpflichtig ist. Der Client selbst und der Grundbetrieb bleiben kostenlos.
- Ohne festes Passwort ist kein unbeaufsichtigter Zugriff möglich, was für betreuten Support korrekt ist, spontanen Zugriff auf ein unbesetztes Gerät aber verhindert.
- Die Funktionsbreite und Stabilität einzelner Plattformen, besonders auf mobilen Geräten, erreichen die kommerziellen Produkte nicht in jedem Detail. Prüfen Sie die für Sie wichtigen Funktionen vor dem Umstieg.
- Manche Sicherheitsprogramme melden Fernwartungssoftware als potenziell unerwünscht. Pflegen Sie bei Bedarf eine Ausnahme und dokumentieren Sie, warum die Software installiert ist.

Für den privaten Gebrauch und den Support einzelner Geräte reicht die kostenlose Fassung mit dem öffentlichen Server oder einer Direktverbindung. Sobald Sie viele Geräte verwalten, gewerblich arbeiten oder volle Datenhoheit brauchen, benötigen Sie einen eigenen Server, mit dem entsprechenden Betriebsaufwand als Gegenleistung für die Unabhängigkeit.

## Quellen

1.  [RustDesk auf GitHub](https://github.com/rustdesk/rustdesk): Quellcode, Releases mit den Installern und die Lizenz AGPL-3.0.

2.  [RustDesk-Dokumentation](https://rustdesk.com/docs/): Installation, eigener Server, Ports und Konfiguration der Clients.

3.  [rustdesk-server auf GitHub](https://github.com/rustdesk/rustdesk-server): Server-Komponenten `hbbs` und `hbbr` inklusive der Portübersicht für den Eigenbetrieb.
