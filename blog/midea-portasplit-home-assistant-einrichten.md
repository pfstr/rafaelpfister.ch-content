---
title: "Midea PortaSplit in Home Assistant einbinden und absichern"
navTitle: "PortaSplit einrichten"
description: "Der praktische Teil zur Midea PortaSplit in Home Assistant: welche Community-Integration passt, die Einrichtung Schritt für Schritt, sinnvolle Automationen und eine vollständige Absicherung im Heimnetz mit IoT-VLAN, Firewall-Regeln und verschlüsseltem Backup."
date: "2026-07-24"
kategorie: "Smart Home & IoT"
timeToRead: "14 min to read"
themen:
  - "smart-home-iot"
related:
  - "midea-portasplit-home-assistant"
  - "serverloser-newsletter-cloudflare-workers-d1"
slug: "midea-portasplit-home-assistant-einrichten"
url: "https://rafaelpfister.ch/blog/midea-portasplit-home-assistant-einrichten"
aiPrompt: |
  Du bist mein Smart-Home-Assistent. Hilf mir, eine Midea PortaSplit sicher in Home Assistant zu integrieren:
  1. Gerät zuerst regulär mit der MSmartHome-App ins 2,4-GHz-WLAN aufnehmen.
  2. HACS installieren und die Integration `Midea Smart AC` (mill1000/midea-ac-py) hinzufügen.
  3. Gerät automatisch oder manuell (Device ID, IP, Port 6444, Gerätetyp, Token, Key) einbinden.
  4. Token, Key und die Integrationskonfiguration verschlüsselt ausserhalb von Home Assistant sichern.
  5. DHCP-Reservation setzen, das Gerät in ein separates IoT-VLAN verschieben und Firewall-Regeln so formulieren, dass nur Home Assistant zugreifen darf.
  6. Ausgehenden Internetzugriff testweise blockieren und über mehrere Neustarts prüfen, ob die lokale Steuerung stabil bleibt.
  Warne mich vor jedem Schritt, der die bestehende Kopplung zerstören und eine erneute Token-Beschaffung über die Midea-Cloud erforderlich machen könnte.
---

Dieser Beitrag ist der praktische Teil zur Midea PortaSplit in Home Assistant. Warum die lokale Steuerung überhaupt einen Token und Key aus der Midea-Cloud braucht und weshalb diese Zugangsdaten gerade unter Zeitdruck stehen, klärt der [erste Teil zu lokaler Steuerung und der Cloud-Token-Frage](/blog/midea-portasplit-home-assistant). Hier geht es um die konkrete Umsetzung: die Wahl der Integration, die Einrichtung Schritt für Schritt, sinnvolle Automationen und die Absicherung im eigenen Netz.

Ein Hinweis vorweg: Die beschriebenen Integrationen stammen aus der Community und werden weder von Midea noch von Home Assistant offiziell unterstützt. Firmware-Updates, Änderungen an der Midea-Cloud oder an den Integrationen selbst können das Verhalten jederzeit beeinflussen.

## Welche Integration passt

### Midea Smart AC

Das Repository <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> konzentriert sich auf Midea-Klimageräte und verwandte OEM-Modelle und unterstützt die Gerätetypen `0xAC` und `0xCC`. Es bietet lokale Steuerung, grafische Einrichtung, automatische Erkennung, manuelle Einrichtung mit Token und Key sowie eine automatische Abfrage der Gerätefähigkeiten. Der „Out Silent Mode" der PortaSplit wird explizit unterstützt.

Als Indiz für Kompatibilität nennt das Projekt unter anderem die Apps Artic King, Midea Air, NetHome Plus, SmartHome beziehungsweise MSmartHome, Toshiba AC NA und 美的美居. Die PortaSplit verwendet in Europa typischerweise MSmartHome und passt damit in dieses Ökosystem.

### Midea AC LAN

Das Repository <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> unterstützt nicht nur Klimaanlagen, sondern zahlreiche weitere Midea-Geräteklassen: Luftentfeuchter, Ventilatoren, Luftreiniger, Waschmaschinen, Trockner, Geschirrspüler, Warmwassergeräte, Wärmepumpen, Kühlschränke und mehr, teils auch unter Fremdmarken wie Carrier oder Electrolux. Es bietet ebenfalls lokale Kommunikation, automatische Geräteerkennung und zusätzliche Sensoren und hält laut Projektbeschreibung eine längere TCP-Verbindung zum Gerät offen, um Statusänderungen zeitnah zu synchronisieren. Vorausgesetzt wird mindestens Home Assistant 2024.4.1.

Der grösste Nachteil ist derzeit die Warnung des Entwicklers: Die für das Hinzufügen neuer Geräte verwendeten Cloud-Token-APIs werden schrittweise abgeschaltet. Das spätere Hinzufügen neuer Geräte kann dadurch unmöglich werden. Die Einordnung dazu steht im [ersten Teil](/blog/midea-portasplit-home-assistant#die-warnung-von-midea-ac-lan).

### Empfehlung

Für eine reine PortaSplit-Installation würde ich mit `Midea Smart AC` beginnen und `Midea AC LAN` als Alternative im Hinterkopf behalten. `Midea Smart AC` ist enger auf Klimageräte zugeschnitten und dokumentiert die aktuellen PortaSplit-Funktionen explizit.

Beide Integrationen gleichzeitig und dauerhaft mit demselben Gerät zu betreiben ist nicht sinnvoll. Mehrere parallele Verbindungen führen zu Statusproblemen, unnötigem Netzwerkverkehr und schwer nachvollziehbarem Verhalten.

## Was die Integration bringt

Nach der Einrichtung erscheint die PortaSplit als `climate`-Entität in Home Assistant. Je nach Firmware und Integration stehen unter anderem folgende Funktionen zur Verfügung:

- Ein- und Ausschalten
- Solltemperatur einstellen
- aktuelle Raumtemperatur auslesen
- Kühlen, Entfeuchten und reiner Lüfterbetrieb
- Lüftergeschwindigkeit einstellen
- Swing-Funktion steuern
- Eco- und Boost-Modus
- Luftfeuchtigkeit auslesen
- Fehlercodes anzeigen
- Energie- und Leistungswerte auslesen
- Kompressorwerte anzeigen
- Leisemodus des Aussengeräts aktivieren

Welche Entitäten tatsächlich erscheinen, hängt vom Modell, von der Firmware, vom verwendeten Protokoll und von der jeweiligen Integration ab. `Midea Smart AC` fragt die vom Gerät gemeldeten Fähigkeiten ab und blendet Funktionen aus, die das Modell nicht unterstützt. `Midea AC LAN` dokumentiert ebenfalls umfangreiche Klimaentitäten, darunter Temperatur, Luftfeuchtigkeit, aktuelle Leistung, Gesamtenergie, Kompressorfrequenz, Pumpenstatus und verschiedene Betriebsmodi, und nennt für bestimmte PortaSplit-Untertypen eigene Methoden zur Dekodierung der Energiedaten.

Nicht jede angezeigte Messung muss korrekt sein. Gerade Energieverbrauch und Leistung werden bei unterschiedlichen Midea-Modellen in verschiedenen Formaten übertragen. Zeigt Home Assistant offensichtlich falsche Werte an, ist meist die verwendete Dekodierungsmethode anzupassen und nicht das Gerät defekt.

## Voraussetzungen

Benötigt werden eine Midea PortaSplit mit WLAN-Funktion, ein 2,4-GHz-WLAN, die MSmartHome-App, ein Midea-Benutzerkonto, Home Assistant, HACS und Netzwerkzugriff zwischen Home Assistant und PortaSplit. Die PortaSplit sollte zuerst regulär mit der MSmartHome-App verbunden werden, erst danach wird sie in Home Assistant aufgenommen.

## Schritt 1: PortaSplit mit MSmartHome verbinden

1. MSmartHome-App installieren.
2. Midea-Konto erstellen oder anmelden.
3. PortaSplit in den WLAN-Kopplungsmodus versetzen.
4. Gerät mit dem 2,4-GHz-WLAN verbinden.
5. Prüfen, ob sich die PortaSplit über die App steuern lässt.

Viele IoT-Geräte unterstützen weiterhin nur 2,4 GHz. Nutzt der Router dieselbe SSID für 2,4 und 5 GHz, funktioniert die Einrichtung meist trotzdem. Bei Problemen hilft es, vorübergehend ein separates 2,4-GHz-WLAN bereitzustellen.

## Schritt 2: HACS installieren

HACS ist der Home Assistant Community Store. Damit lassen sich Community-Integrationen installieren, die nicht Bestandteil von Home Assistant Core sind. Nach der HACS-Installation wird HACS geöffnet, zu den Integrationen gewechselt, nach `Midea Smart AC` gesucht, die Integration heruntergeladen und Home Assistant neu gestartet. Alternativ kann nach `Midea AC LAN` gesucht werden.

HACS vereinfacht Installation und Updates. Es macht eine Custom Integration jedoch nicht zu einer offiziell geprüften Home-Assistant-Komponente. Dieser Unterschied ist aus Security-Sicht wesentlich und weiter unten Thema.

## Schritt 3: Midea Smart AC hinzufügen

Nach dem Neustart geht es über Einstellungen, Geräte & Dienste und Integration hinzufügen zur Suche nach `Midea Smart AC`, dort dann zu `Discover devices`. Die Integration kann entweder das gesamte lokale Netz durchsuchen oder gezielt die IP-Adresse der PortaSplit abfragen.

Wird das Gerät gefunden, benötigt die Integration bei neueren V3-Geräten Region, Midea-Konto, Passwort und Geräte-ID sowie den daraus abgeleiteten Token und Key. Die Cloud-Region muss zum verwendeten Konto passen. Bei Problemen empfiehlt das Projekt, auch die anderen angebotenen Regionen auszuprobieren.

### Manuelle Einrichtung

Scheitert die automatische Einrichtung, lässt sich das Gerät manuell konfigurieren. Bei `Midea Smart AC` werden dafür folgende Angaben benötigt:

```text
Device ID
IP-Adresse
Port
Gerätetyp
Token
Key
```

Der dokumentierte Standardport ist:

```text
6444/TCP
```

Für V3-Geräte gibt die Dokumentation den Token als 128-stellige und den Key als 64-stellige Hexadezimalzeichenkette an. Beide Werte sind Geheimnisse und entsprechend zu behandeln. Wer die Zugangsdaten nicht über die Discovery beziehen möchte, kann sie mit dem eigenen Konto über die CLI `msmart-ng` abrufen.

## Die PortaSplit sicher betreiben

Wer die PortaSplit lokal steuert, holt einen Teil der Kontrolle aus der Hersteller-Cloud zurück, verlagert die Verantwortung damit aber ins eigene Netz. Die folgenden Punkte sorgen dafür, dass Token und Key auch bei einer Panne wenig Schaden anrichten und das Gerät sauber isoliert bleibt.

### Token und Key sind Geheimnisse

Token und Key authentifizieren die lokale Kommunikation mit dem Gerät und sind wie ein Passwort zu behandeln. Warum sie so heikel sind und was ein Angreifer mit ihnen anfangen könnte, steht im [ersten Teil](/blog/midea-portasplit-home-assistant#was-das-für-die-sicherheit-bedeutet). Für den Betrieb zählt vor allem: Sie gehören nicht in Logs, nicht in unverschlüsselte Backups und nicht in ein Repository.

### Kein Portforwarding zur PortaSplit

Der häufigste vermeidbare Fehler wäre, den lokalen Geräteport direkt aus dem Internet erreichbar zu machen. Eine Regel wie diese wäre gefährlich:

```text
Internet → TCP 6444 → PortaSplit
```

Es gibt keinen guten Grund, die PortaSplit direkt aus dem Internet erreichbar zu machen. Home Assistant befindet sich bereits im lokalen Netz und dient als kontrollierende Instanz. Der Router sollte keine Portweiterleitung zur PortaSplit besitzen, UPnP nach Möglichkeit einschränken oder deaktivieren, eingehende Verbindungen standardmässig blockieren und keine DMZ-Freigabe für das Gerät verwenden.

### Eigenes IoT-VLAN

Die beste Netzwerkarchitektur ist ein separates IoT-Netz:

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

Die PortaSplit befindet sich im IoT-VLAN. Home Assistant darf gezielt auf das Gerät zugreifen, die PortaSplit darf aber nicht beliebig auf PCs, NAS und andere interne Systeme zugreifen. Eine mögliche Firewall-Logik:

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Während der erstmaligen Einrichtung benötigt das Gerät Internetzugriff zur Midea-Cloud. Nach erfolgreicher lokaler Einrichtung lässt sich testen, ob der ausgehende Internetzugriff blockiert werden kann. Dabei sollte nicht sofort eine endgültige Sperre gesetzt werden. Zuerst ist zu prüfen, ob die lokale Steuerung weiterhin funktioniert, ob das Gerät nach einem Neustart erreichbar bleibt, ob es einen Router-Neustart übersteht, ob es auch nach mehreren Tagen noch reagiert, ob die MSmartHome-App weiterhin benötigt wird und ob Firmware-Updates noch angeboten werden. Wer Cloud und Firmware-Updates weiter nutzen möchte, kann ausgehenden Internetzugriff zeitweise erlauben und danach wieder blockieren.

### Netzwerksegmentierung kann Discovery verhindern

Automatische Gerätesuche basiert häufig auf Broadcast- oder Multicast-Verkehr, und der wird normalerweise nicht über VLAN-Grenzen geroutet. Home Assistant findet die PortaSplit deshalb möglicherweise nicht automatisch, obwohl eine reguläre IP-Verbindung erlaubt wäre.

Dann hilft es, die PortaSplit vorübergehend im selben VLAN wie Home Assistant einzurichten, die Geräte-IP manuell anzugeben, eine geeignete Broadcast-Relay-Funktion zu verwenden oder nach der Einrichtung gezielte Firewall-Regeln zu definieren. Die manuelle Konfiguration ist aus Security-Sicht häufig sogar die bessere Variante, weil dafür kein zusätzlicher Broadcast-Verkehr zwischen den Netzen erlaubt werden muss.

### Statische DHCP-Zuordnung

Die PortaSplit sollte im Router eine feste DHCP-Zuordnung erhalten:

```text
PortaSplit → 192.168.30.25
```

Eine DHCP-Reservation ist einer im Gerät gesetzten statischen IP meist vorzuziehen. Home Assistant findet das Gerät zuverlässig, Firewall-Regeln lassen sich auf eine feste Adresse beschränken, die Fehleranalyse wird einfacher, und nach Router- oder Geräte-Neustarts bleibt die Zuordnung stabil. Eine Firewall-Regel kann damit sehr eng formuliert werden:

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

Der tatsächlich benötigte Port ist anhand der Integration und des eigenen Geräts zu verifizieren.

### Home Assistant als zentraler Vertrauensanker

Wer die PortaSplit lokal steuert, verlagert das Vertrauen teilweise von der Midea-Cloud zu Home Assistant. Wird Home Assistant kompromittiert, kontrolliert ein Angreifer unter Umständen nicht nur die Klimaanlage, sondern das gesamte Smart Home.

Home Assistant sollte deshalb regelmässig aktualisiert werden, nicht per ungeschützter Portweiterleitung veröffentlicht sein, mit einem starken, einzigartigen Passwort geschützt sein, Mehrfaktor-Authentifizierung verwenden, verschlüsselte Backups erstellen, nur notwendige Add-ons enthalten und keinen unnötigen SSH-Zugang aus dem Internet erlauben. Für den Fernzugriff sind ein VPN, Home Assistant Cloud oder ein sauber konfigurierter Reverse Proxy die besseren Optionen als eine simple Portweiterleitung auf Port 8123.

### HACS und das Supply-Chain-Risiko

`Midea Smart AC` und `Midea AC LAN` sind Custom Integrations. Sie laufen innerhalb von Home Assistant und erhalten damit weitreichenden Zugriff auf dessen Laufzeitumgebung. Eine bösartige oder kompromittierte Integration könnte theoretisch Konfigurationsdaten lesen, Secrets auslesen, Netzwerkverbindungen aufbauen, Geräte im lokalen Netzwerk scannen, Zustände anderer Entitäten lesen, Daten an externe Systeme übertragen und die Verfügbarkeit von Home Assistant beeinträchtigen.

Das heisst nicht, dass die genannten Integrationen bösartig sind. Beide Projekte sind öffentlich einsehbar, werden aktiv entwickelt und haben eine sichtbare Community. Open Source ist jedoch keine automatische Sicherheitsgarantie. Vor der Installation lohnt sich mindestens der Blick darauf, ob das Repository aktiv gepflegt wird, ob es regelmässige Releases gibt, wie viele Personen zum Code beitragen, ob offene Security-Issues bestehen, ob kürzlich Maintainer oder Repository-Eigentümer gewechselt haben, ob HACS auf das erwartete Repository verweist und ob ein Update ungewöhnlich grosse oder unerklärliche Änderungen enthält.

Updates sollten nicht blind unmittelbar nach Veröffentlichung installiert werden. Gerade bei sicherheitskritischen Smart-Home-Systemen ist es sinnvoll, einige Tage zu warten und Release Notes sowie gemeldete Probleme zu prüfen.

### Cloud-Konto absichern

Solange die Midea-Cloud für die Einrichtung oder die App-Steuerung verwendet wird, bleibt auch das Midea-Konto Teil des Sicherheitsmodells. Es gehört ein einzigartiges Passwort dazu, das nicht mit anderen Diensten geteilt wird, ein Passwortmanager, Mehrfaktor-Authentifizierung sofern angeboten, das Entfernen alter Smartphones und Sitzungen, der Verzicht auf gemeinsam genutzte Konten und eine regelmässige Kontrolle, welche Geräte im Konto registriert sind.

Verlangt die Home-Assistant-Integration während der Einrichtung Benutzername und Passwort, ist zu prüfen, ob die Zugangsdaten nur für den einmaligen Token-Abruf oder dauerhaft gespeichert werden. Die Entwickler von `Midea Smart AC` schreiben, dass Geräte nach der Einrichtung nicht mit eingebauten Integrationskonten verknüpft werden und dass Token und Key auch manuell über das eigene Konto per CLI beschafft werden können. Wo möglich, ist das eigene Konto gegenüber fremden oder integrierten Sammelkonten vorzuziehen.

### Cloud blockieren oder nicht?

Nach erfolgreicher Einrichtung stellt sich die Frage, ob der Internetzugriff der PortaSplit vollständig blockiert werden sollte. Für eine Sperre sprechen weniger Telemetrie, eine geringere Abhängigkeit von externen Diensten, ein kleinerer Angriffsweg über die Hersteller-Cloud, die Tatsache, dass das Gerät keine beliebigen externen Ziele kontaktieren kann, und die geringere Wirkung cloudseitiger Änderungen.

Dagegen spricht, dass die MSmartHome-App ausserhalb des Heimnetzes möglicherweise nicht mehr funktioniert, dass Firmware-Updates nicht mehr geladen werden, dass Uhrzeit- oder Cloud-Funktionen ausfallen können, dass eine erneute Anmeldung oder Wiederherstellung schwieriger wird und dass manche Geräte nach längerer Offline-Zeit unerwartet reagieren.

Eine pragmatische Reihenfolge: Gerät normal einrichten, Home Assistant und App testen, Token und Konfiguration sichern, Internetzugriff sperren, Gerät und Home Assistant neu starten, mehrere Tage beobachten und bei Bedarf den Internetzugriff nur temporär wieder freigeben.

### Firmware-Updates: Sicherheitsgewinn oder Integrationsrisiko?

Firmware-Updates sind bei IoT-Geräten ein Dilemma. Sie können bekannte Schwachstellen schliessen, die Stabilität verbessern, Sicherheitsmechanismen modernisieren und neue Funktionen bringen. Sie können aber auch lokale Schnittstellen ändern, Reverse-Engineering-Integrationen brechen, Token ungültig machen, die lokale API deaktivieren und neue Cloud-Abhängigkeiten einführen.

Die im Januar 2026 ausgelieferte PortaSplit-Firmware brachte beispielsweise einen neuen Leisemodus für das Aussengerät, der die Geräuschentwicklung um rund 6 Dezibel senkt. Dieser musste von den Community-Integrationen erst nachvollzogen und implementiert werden, dokumentiert in einem eigenen GitHub-Issue für die PortaSplit.

Daraus folgt: Firmware-Updates nicht grundsätzlich verhindern, vor einem Update prüfen, ob andere Home-Assistant-Nutzer Probleme melden, Konfiguration und Token vorher sichern, ein Home-Assistant-Backup erstellen und nach dem Update die lokale Steuerung vollständig testen. Security bedeutet nicht „nie aktualisieren". Veraltete Firmware kann gefährlicher sein als eine vorübergehend inkompatible Integration.

### Debug-Logs enthalten sensible Daten

Bei Problemen verlangen Open-Source-Projekte häufig Debug-Logs. Die Dokumentation von `Midea AC LAN` zeigt, wie das Logging für die beiden relevanten Komponenten aktiviert wird:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Danach lassen sich die Logs über Einstellungen, System und Protokolle herunterladen. Solche Logs können je nach Integration und Fehlerfall lokale IP-Adressen, Geräte-ID, Seriennummer, Modellkennung, Cloud-Antworten, Accountinformationen, Token oder Teile davon, Netzwerkpakete sowie Zeitstempel und Nutzungsverhalten enthalten. Vor dem Hochladen in ein öffentliches GitHub-Issue sind sie daher zu prüfen und sensible Werte zu schwärzen.

Nach Abschluss der Fehlersuche gehört das Debug-Logging wieder entfernt. Dauerhaft aktiviertes Debug-Logging erhöht nicht nur den Speicherverbrauch, es vergrössert auch die Menge sensibler Informationen in den Backups.

### Was Midea selbst zur Sicherheit sagt

Midea wirbt für sein SmartHome-Ökosystem mit der Orientierung an mehreren Sicherheits- und Datenschutzstandards, genannt werden EN 303 645, UK PSTI, NIST, DSGVO-konforme Datenverarbeitung und die Anforderungen der EU Radio Equipment Directive. Das sind positive Signale, aber keine Aussage darüber, wie jede einzelne PortaSplit-Firmware, jeder Cloud-Endpunkt und jede lokale API tatsächlich implementiert ist. Zertifizierungs- und Marketingaussagen ersetzen keine technische Prüfung des konkreten Geräts.

Genauso wäre es falsch, aus der Warnung einer Community-Integration abzuleiten, die PortaSplit sei generell unsicher. Das beschriebene Problem betrifft die Architektur langlebiger Tokens und deren Verwendung durch inoffizielle Clients.

### Risiko nach Szenario

| Szenario | Risiko | Begründung |
| --- | --- | --- |
| Normales Heimnetz ohne Portweiterleitung | überschaubar | Ein Angreifer braucht zuerst Zugriff auf WLAN, Home Assistant oder ein Backup. |
| Flaches Heimnetz mit vielen unsicheren IoT-Geräten | mittel | Ein kompromittiertes anderes IoT-Gerät kann PortaSplit oder Home Assistant im selben Netz erreichen. |
| PortaSplit direkt aus dem Internet erreichbar | hoch | Das Gerät sollte niemals per Portweiterleitung veröffentlicht werden. |
| Token und Key öffentlich auf GitHub | hoch | Die Geheimnisse gelten als kompromittiert; ob sie widerrufen werden können, ist nicht garantiert. |
| Separates IoT-VLAN, restriktive Firewall, lokale Steuerung | vergleichsweise gering | Selbst bei einer Schwachstelle im Gerät ist die Bewegungsfreiheit im Netz stark eingeschränkt. |

## Backup der Konfiguration

Die Sicherung von Token, Key und Konfiguration ist der wichtigste einmalige Schritt: Sind die Cloud-Token-Schnittstellen erst geschlossen, ist ein Backup der einzige Weg zu einer Neueinrichtung. `Midea AC LAN` legt nach erfolgreicher Einrichtung für V3-Geräte eine JSON-Konfigurationsdatei ab. Der dokumentierte Pfad lautet:

```text
/config/.storage/midea_ac_lan/
```

Die Datei trägt die Geräte-ID als Dateinamen:

```text
<device-id>.json
```

Diese Datei ist keine normale Textnotiz. Sie kann Geräte-ID, Seriennummer, IP-Adresse, Token, Key, Protokollinformationen sowie Cloud- und Geräteparameter enthalten. Entsprechend gilt:

- Nicht in ein öffentliches GitHub-Repository hochladen.
- Nicht in Foren posten.
- Nicht als ungeschwärzten Screenshot teilen.
- Nicht per unverschlüsselter E-Mail verschicken.

Auch ein privates Git-Repository ist nicht automatisch der richtige Speicherort, weil Geheimnisse in der Git-Historie verbleiben, selbst wenn sie später aus der aktuellen Datei gelöscht werden. Geeigneter sind ein verschlüsseltes Backup, ein Passwortmanager mit Dateianhang, ein verschlüsseltes NAS-Backup, ein verschlüsseltes Offline-Medium oder ein verschlüsseltes Archiv mit separat gespeichertem Passwort.

Zur Sicherung über das Home-Assistant-Terminal:

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Datei anzeigen:

```bash
cat <device-id>.json
```

Zum Kopieren sollte die Datei nicht über einen öffentlichen Webdienst übertragen werden. Besser ist ein verschlüsseltes Archiv, das anschliessend in ein verschlüsseltes Backup überführt wird:

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

Die Dateien in `.storage` sollten nicht manuell bearbeitet werden. Der Entwickler empfiehlt ausdrücklich, die JSON-Datei bei Problemen weder zu löschen noch direkt zu verändern, sondern sie vor Änderungen umzubenennen und zu sichern.

Ein vollständiges Home-Assistant-Backup enthält diese Dateien ebenfalls. Eine separate Kopie ist dennoch sinnvoll, weil Home-Assistant-Backups beschädigt werden können, ein Restore die Integration überschreiben kann, die Datei gezielt für eine spätere Neueinrichtung benötigt werden könnte und ein Backup nie nur auf demselben System liegen sollte.

## Secrets aus einem veröffentlichten Git-Repository entfernen

Wurde eine JSON-Datei versehentlich auf GitHub veröffentlicht, reicht normales Löschen und ein neuer Commit nicht aus. Die Datei bleibt in der Git-Historie abrufbar. Mindestens diese Schritte sind nötig:

1. Repository sofort auf privat stellen, sofern möglich.
2. Datei aus der gesamten Git-Historie entfernen.
3. GitHub-Caches und Forks berücksichtigen.
4. Token als kompromittiert behandeln.
5. Gerät aus dem Midea-Konto entfernen und neu verbinden, falls dadurch neue Schlüssel erzeugt werden.
6. Home-Assistant-Integration neu einrichten.
7. Midea-Kontopasswort ändern, falls Zugangsdaten ebenfalls betroffen waren.

Ob das erneute Pairing tatsächlich einen neuen Token erzeugt, variiert je nach Gerät und Cloud-Architektur. Darauf, dass das Ändern des Kontopassworts automatisch den lokalen Gerätetoken ungültig macht, sollte man sich nicht verlassen.

## Sinnvolle Automationen

Nach erfolgreicher Integration lässt sich die PortaSplit deutlich intelligenter betreiben. Die Entity-IDs sind an die eigene Installation anzupassen.

Nur bei geschlossenen Fenstern kühlen:

```yaml
alias: PortaSplit nur bei geschlossenen Fenstern
triggers:
  - trigger: state
    entity_id: binary_sensor.wohnzimmer_fenster
    to: "on"

actions:
  - action: climate.turn_off
    target:
      entity_id: climate.portasplit
```

Bei hoher Raumtemperatur einschalten:

```yaml
alias: PortaSplit bei Hitze einschalten
triggers:
  - trigger: numeric_state
    entity_id: sensor.wohnzimmer_temperatur
    above: 27

conditions:
  - condition: state
    entity_id: binary_sensor.wohnzimmer_fenster
    state: "off"
  - condition: state
    entity_id: person.rafael
    state: "home"

actions:
  - action: climate.set_hvac_mode
    target:
      entity_id: climate.portasplit
    data:
      hvac_mode: cool

  - action: climate.set_temperature
    target:
      entity_id: climate.portasplit
    data:
      temperature: 24
```

Vor dem Schlafengehen vorkühlen:

```yaml
alias: Schlafzimmer vorkühlen
triggers:
  - trigger: time
    at: "21:00:00"

conditions:
  - condition: numeric_state
    entity_id: sensor.schlafzimmer_temperatur
    above: 25

actions:
  - action: climate.set_temperature
    target:
      entity_id: climate.portasplit
    data:
      temperature: 23
```

Ausschalten, wenn niemand zu Hause ist:

```yaml
alias: PortaSplit bei Abwesenheit ausschalten
triggers:
  - trigger: state
    entity_id: zone.home
    to: "0"
    for:
      minutes: 10

actions:
  - action: climate.turn_off
    target:
      entity_id: climate.portasplit
```

## Empfohlene Konfiguration im Überblick

```text
1. PortaSplit mit MSmartHome einrichten
2. Midea Smart AC über HACS installieren
3. PortaSplit automatisch oder manuell hinzufügen
4. DHCP-Reservation erstellen
5. Home-Assistant-Backup anfertigen
6. Token- und Konfigurationsdaten verschlüsselt sichern
7. PortaSplit in ein separates IoT-VLAN verschieben
8. Zugriff von Home Assistant zur PortaSplit erlauben
9. Zugriff der PortaSplit auf interne Netze blockieren
10. Internetzugriff testweise blockieren
11. lokale Steuerung nach Neustarts prüfen
12. Firmware- und Integrationsupdates kontrolliert durchführen
```

Die gewünschte Kommunikationsrichtung sieht damit so aus:

```text
Home Assistant
    │
    │ gezielt erlaubt
    ▼
Midea PortaSplit
    │
    ├── kein Zugriff auf PCs
    ├── kein Zugriff auf NAS
    ├── kein Zugriff auf Management-Netz
    └── Internet nur bei Bedarf
```

## Fazit

Die Midea PortaSplit lässt sich erstaunlich gut in Home Assistant integrieren. Nach erfolgreicher Einrichtung ist sie lokal steuerbar und in Automationen einbindbar, womit für den täglichen Betrieb ein grosser Teil der Cloud-Abhängigkeit entfällt.

Unter Sicherheitsgesichtspunkten ist die Integration vertretbar, wenn einige Grundregeln eingehalten werden: keine Portweiterleitung, Token und Key geheim halten, Backups verschlüsseln, Debug-Logs vor Veröffentlichung prüfen, Home Assistant absichern, IoT-Geräte segmentieren, ausgehenden Internetzugriff auf das Notwendige beschränken und Firmware- sowie HACS-Updates nicht blind installieren. So betrieben ist die PortaSplit nicht nur eine leistungsfähige Klimaanlage, sondern ein sinnvoll integrierbarer Bestandteil eines lokal gesteuerten Smart Homes.

Warum die Einrichtung überhaupt einmalig die Midea-Cloud braucht und weshalb dieser Weg unter Zeitdruck steht, steht im [ersten Teil zu lokaler Steuerung und der Cloud-Token-Frage](/blog/midea-portasplit-home-assistant).

## Quellen

1.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> — Integration `Midea Smart AC`: unterstützte Gerätetypen `0xAC` und `0xCC`, PortaSplit mit „Out Silent Mode", Cloud-Nutzung zur Token- und Key-Beschaffung bei V3-Geräten, manuelle Konfiguration und Standardport 6444.

2.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> — Integration `Midea AC LAN`: unterstützte Geräteklassen, längere TCP-Verbindung zur Statussynchronisierung und Mindestversion Home Assistant 2024.4.1.

3.  [midea_ac_lan: Dokumentation der Klimaentitäten](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md) — Entitäten und Attribute für Klimageräte, darunter Leistung, Gesamtenergie, Kompressorfrequenz und die Dekodierungsmethoden für Energiewerte einzelner Untertypen.

4.  [midea_ac_lan: Debug- und Konfigurationshinweise](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md) — Ablage der Gerätekonfiguration unter `/config/.storage/midea_ac_lan/`, Empfehlung zum Sichern statt Löschen der JSON-Datei und die Logger-Konfiguration für Debug-Logs.

5.  [Issue 779: Out Silent Mode der PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/779) — Anfrage zur Unterstützung des mit dem Firmware-Update von Januar 2026 eingeführten Leisemodus des Aussengeräts, der die Geräuschentwicklung um rund 6 Dezibel senkt.

6.  [Midea SmartHome](https://www.midea.com/global/smarthome) — Herstellerangaben zu den Sicherheits- und Datenschutzstandards EN 303 645, PSTI, NIST, DSGVO und RED DA.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/) — Installation und Verwaltung von Custom Integrations, die nicht Bestandteil von Home Assistant Core sind.
