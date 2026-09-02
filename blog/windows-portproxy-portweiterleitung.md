---
title: "Portweiterleitung mit netsh portproxy: interne Dienste über einen Sprunghost erreichen"
navTitle: "netsh portproxy"
description: "Windows bringt mit netsh interface portproxy eine eingebaute TCP-Portweiterleitung mit. Zusammen mit einem VPN wie Tailscale erreichen Sie damit einen internen Dienst, etwa eine NAS-Oberfläche, von aussen, ohne ihn öffentlich zu exponieren. Wie Sie die Weiterleitung einrichten, absichern und wieder entfernen, und wo ihre Grenzen liegen: kein UDP, keine zusätzliche Verschlüsselung, Zertifikats- und Redirect-Fallstricke."
date: "2026-09-02"
kategorie: "Windows und Netzwerk"
timeToRead: "9 Min. Lesezeit"
themen:
  - "windows-client"
produkte:
  - "windows-client"
protokolle:
  - "tcp"
  - "haertung"
slug: "windows-portproxy-portweiterleitung"
translationId: "article-236adcb4ae982572"
url: "https://rafaelpfister.ch/blog/windows-portproxy-portweiterleitung"
aiPrompt: |
  Du bist mein Windows-Administrationsassistent. Hilf mir, mit netsh interface portproxy eine TCP-Portweiterleitung über einen Windows-Sprunghost einzurichten, um einen internen Dienst (z. B. eine NAS-Weboberfläche) über ein VPN zu erreichen: Weiterleitung anlegen, Firewall auf den VPN-Bereich beschränken, prüfen, wieder entfernen, und die Grenzen (kein UDP, keine Verschlüsselung, Zertifikats- und Redirect-Probleme) einordnen.
---
# Portweiterleitung mit netsh portproxy: interne Dienste über einen Sprunghost erreichen

Ein interner Dienst hört oft nur im lokalen Netz: die Weboberfläche eines NAS, ein Drucker-Panel, eine Verwaltungsseite. Wollen Sie darauf von aussen zugreifen, ohne den Dienst ins Internet zu stellen, brauchen Sie einen Weg über einen Rechner, der beide Seiten sieht. Windows bringt dafür ein eingebautes Werkzeug mit: `netsh interface portproxy` leitet eingehende TCP-Verbindungen auf ein anderes Ziel weiter. In Kombination mit einem VPN wie Tailscale oder WireGuard wird ein Rechner im Zielnetz zum Sprunghost, über den Sie den internen Dienst erreichen.

Ein konkretes Beispiel: Ein NAS mit der Weboberfläche auf `10.0.0.245:5000` ist nur im lokalen Netz erreichbar. Im selben Netz steht ein Windows-PC, der zusätzlich per VPN erreichbar ist. Richten Sie auf diesem PC eine Portweiterleitung von seiner VPN-Adresse auf das NAS ein, öffnen Sie die NAS-Oberfläche anschliessend im Browser über die VPN-Adresse des PCs. Der Dienst bleibt im internen Netz, nur der Sprunghost ist über das VPN erreichbar.

## Wie portproxy arbeitet

`portproxy` ist ein Bestandteil des Dienstes IP Helper (`iphlpsvc`). Der Dienst nimmt Verbindungen auf einem lokalen Port an und reicht sie an ein Ziel weiter. Es ist ein reiner TCP-Relay auf Anwendungsebene: keine Firewall-NAT-Regel, sondern ein Prozess, der Bytes zwischen zwei Verbindungen kopiert. Läuft `iphlpsvc` nicht, funktioniert keine Weiterleitung. Der Dienst ist standardmässig vorhanden; sein Starttyp sollte auf automatisch stehen, wenn die Weiterleitung einen Neustart überdauern soll.

## Einrichten

Eine Weiterleitung braucht zwei Schritte: die portproxy-Regel und eine Firewallregel, die den Zugriff auf den Listener erlaubt. Führen Sie beides in einer Eingabeaufforderung oder PowerShell mit Administratorrechten aus.

Zuerst die Weiterleitung. Sie bindet an eine lokale Adresse und einen Port und verweist auf Ziel-IP und Ziel-Port:

```powershell
netsh interface portproxy add v4tov4 listenaddress=100.100.10.10 listenport=5000 connectaddress=10.0.0.245 connectport=5000
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `v4tov4` | IPv4 hört, IPv4 verbindet; ebenso möglich: `v4tov6`, `v6tov4`, `v6tov6` |
| `listenaddress` | Lokale Adresse, auf der gehört wird; hier die VPN-Adresse des Sprunghosts, damit nur über das VPN eingegangen wird |
| `listenport` | Lokaler Port, auf dem gehört wird |
| `connectaddress` | Ziel-IP, an die weitergereicht wird (der interne Dienst) |
| `connectport` | Ziel-Port am internen Dienst |

</details>

Die Bindung an die VPN-Adresse statt an `0.0.0.0` ist die erste Absicherung: Der Listener erscheint nur auf der VPN-Schnittstelle, nicht auf allen Netzwerkkarten des Sprunghosts. Die zweite Absicherung ist die Firewall. Öffnen Sie den Listener-Port ausschliesslich für den Adressbereich Ihres VPN, nicht für alle Adressen:

```powershell
New-NetFirewallRule -DisplayName "NAS-Proxy (VPN)" -Direction Inbound -Action Allow -Protocol TCP -LocalPort 5000 -RemoteAddress 100.64.0.0/10
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `-Direction Inbound` | Regel für eingehenden Verkehr |
| `-Protocol TCP` | portproxy leitet nur TCP weiter, darum TCP |
| `-LocalPort 5000` | Der Listener-Port aus der portproxy-Regel |
| `-RemoteAddress 100.64.0.0/10` | Nur Quellen aus diesem Bereich sind zugelassen; hier der Tailscale-Bereich, sonst der CIDR-Block Ihres VPN |

</details>

## Prüfen und nutzen

Prüfen Sie zuerst auf dem Sprunghost selbst, ob der interne Dienst überhaupt erreichbar ist, und lassen Sie sich die aktive Weiterleitung anzeigen:

```powershell
Test-NetConnection -ComputerName 10.0.0.245 -Port 5000
netsh interface portproxy show v4tov4
```

Antwortet das Ziel und steht die Regel in der Liste, testen Sie von Ihrem entfernten Gerät aus. Erreichbar ist der Dienst jetzt über Adresse und Port des Sprunghosts:

```powershell
Test-NetConnection -ComputerName 100.100.10.10 -Port 5000
```

Im Browser öffnen Sie dann `http://100.100.10.10:5000`. Brauchen Sie mehrere Ports desselben Dienstes, etwa 5000 und 5001 für http und https, legen Sie für jeden Port eine eigene portproxy-Regel und die passende Firewallfreigabe an.

## Manpage-artige Übersicht

Die wichtigsten Unterbefehle von `netsh interface portproxy`:

<details class="options-details">
<summary>Optionen im Überblick</summary>

| Befehl | Zweck |
|---|---|
| `add v4tov4 …` | Weiterleitung anlegen (listenaddress/listenport → connectaddress/connectport) |
| `show v4tov4` | Aktive IPv4-Weiterleitungen anzeigen |
| `show all` | Alle Weiterleitungen aller Protokollvarianten anzeigen |
| `delete v4tov4 listenaddress=… listenport=…` | Eine Weiterleitung entfernen |
| `reset` | Alle portproxy-Regeln löschen |

</details>

Die Regeln liegen in der Registry unter `HKLM\SYSTEM\CurrentControlSet\Services\PortProxy` und überstehen einen Neustart. Sichtbar sind sie nur über `netsh` oder direkt in der Registry, nicht in der grafischen Firewall-Oberfläche.

## Alternativen

`portproxy` ist praktisch, wenn der Sprunghost ohnehin Windows ist und Sie nichts nachinstallieren wollen. Zwei Alternativen lösen dasselbe Problem mit anderen Eigenschaften.

Ein SSH-Tunnel mit lokaler Weiterleitung (`ssh -L 5000:10.0.0.245:5000 benutzer@sprunghost`) verschlüsselt die Strecke bis zum Sprunghost selbst und läuft plattformübergreifend. Er braucht einen SSH-Server auf dem Sprunghost und besteht nur, solange die SSH-Sitzung läuft.

Ein Tailscale-Subnetz-Router (`tailscale up --advertise-routes=10.0.0.0/24`) macht das gesamte interne Subnetz für Ihre VPN-Geräte erreichbar. Dann adressieren Sie den internen Dienst direkt unter seiner echten IP, ohne Weiterleitung pro Port. Das ist der geradlinigste Weg, wenn Sie mehrere interne Geräte erreichen wollen, verlangt aber eine Freigabe der Route in der Tailscale-Verwaltung.

## Grenzen

Eine Portweiterleitung mit portproxy löst den Zugriff, aber sie hat klare Grenzen, die Sie vor dem Einsatz kennen sollten:

- **Nur TCP.** `portproxy` leitet ausschliesslich TCP weiter. Dienste, die UDP brauchen (DNS, viele VPN- und Spielprotokolle, manche Videoübertragung), lassen sich damit nicht abbilden.
- **Keine zusätzliche Verschlüsselung.** Die Weiterleitung kopiert Bytes unverändert. Die Vertraulichkeit der Strecke liefert allein das VPN, über das Sie den Sprunghost erreichen. Über ein unverschlüsseltes Transportnetz wäre der Verkehr ungeschützt.
- **Zertifikatswarnung bei HTTPS über die IP.** Leiten Sie einen HTTPS-Dienst weiter und rufen ihn über die IP des Sprunghosts auf, passt das Zertifikat des Ziels nicht zur aufgerufenen Adresse. Der Browser warnt. Für einen kurzen Test ist das hinnehmbar, für den Dauerbetrieb nicht.
- **Umleitungen und absolute Adressen.** Manche Weboberflächen leiten selbst auf ihren Hostnamen oder einen anderen Port um oder bauen absolute Links mit ihrer internen Adresse. Dann bricht der Zugriff über den Sprunghost, obwohl die Weiterleitung steht. Solche Dienste brauchen einen echten Reverse-Proxy statt eines reinen Portrelays.
- **Bindung an eine Adresse, die beim Start existieren muss.** Bindet die Regel an eine bestimmte `listenaddress`, muss diese Adresse beim Start des Dienstes vorhanden sein. Kommt die VPN-Schnittstelle erst später hoch, kann die Bindung fehlschlagen, bis der Dienst oder die Regel neu gesetzt wird.
- **Ein zusätzlicher Weg ins interne Netz.** Jede Weiterleitung ist ein Pfad von aussen zu einem internen Dienst. Beschränken Sie die Firewall eng auf den VPN-Bereich, binden Sie an die VPN-Adresse, und entfernen Sie die Weiterleitung, sobald Sie sie nicht mehr brauchen.

## Wieder entfernen

Löschen Sie nach getaner Arbeit die Weiterleitung und die Firewallregel:

```powershell
netsh interface portproxy delete v4tov4 listenaddress=100.100.10.10 listenport=5000
Remove-NetFirewallRule -DisplayName "NAS-Proxy (VPN)"
```

Eine Portweiterleitung ist ein Werkzeug für den gezielten, zeitlich begrenzten Zugriff, nicht für einen dauerhaft offenen Kanal. Für den Dauerbetrieb eines internen Dienstes über das Internet ist ein Reverse-Proxy mit gültigem Zertifikat oder ein VPN mit Subnetz-Routing die sauberere Lösung.

## Quellen

1.  [netsh interface portproxy (Microsoft Learn)](https://learn.microsoft.com/en-us/windows-server/networking/technologies/netsh/netsh-interface-portproxy): Referenz der Unterbefehle, Protokollvarianten und der Abhängigkeit vom IP-Helper-Dienst.

2.  [New-NetFirewallRule (Microsoft Learn)](https://learn.microsoft.com/en-us/powershell/module/netsecurity/new-netfirewallrule): Parameter der Firewallregel, inklusive Einschränkung auf Adressbereiche über RemoteAddress.

3.  [Tailscale: Subnet routers](https://tailscale.com/kb/1019/subnets): Ein ganzes Subnetz über das VPN erreichbar machen, als Alternative zur Weiterleitung pro Port.
