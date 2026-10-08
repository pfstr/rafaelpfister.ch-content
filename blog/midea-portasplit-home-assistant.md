---
title: "Midea PortaSplit in Home Assistant absichern: Token, Key und Heimnetz"
navTitle: "PortaSplit absichern"
description: "Token und Key der PortaSplit stammen aus der Midea-Cloud und laufen nie ab. So sichern Sie die Werte, schotten das Gerät im Heimnetz ab und halten Home Assistant, Integration und Firmware kontrolliert aktuell."
date: "2026-07-24"
kategorie: "Home Assistant und IoT"
timeToRead: "14 Min. Lesezeit"
themen:
  - "smart-home-iot"
produkte:
  - "home-assistant"
protokolle:
  - "apis"
  - "tcp"
related:
  - "midea-portasplit-home-assistant-einrichten"
  - "midea-v2-cloud-api-portasplit-home-assistant"
image: "../images/midea-portasplit-home-assistant/portasplit-dashboard-simuliert.png"
slug: "midea-portasplit-home-assistant"
translationId: "article-a02e26cce22063f1"
url: "https://rafaelpfister.ch/blog/midea-portasplit-home-assistant"
---

<aside class="article-update">
  <p class="article-update__label">Was PortaSplit-Besitzer jetzt tun sollten</p>
  <p>Home Assistant bezieht Token und Key der PortaSplit bei der Einrichtung über private Cloud-Schnittstellen. Das Projekt Midea AC LAN warnt seit dem 19. Mai 2025 vor möglichen Änderungen; ein Abschalttermin des Herstellers ist nicht dokumentiert. Für Besitzer heisst das:</p>
  <ol>
    <li><strong>Token, Key und Konfiguration verschlüsselt sichern.</strong> Falls der Abruf später nicht mehr funktioniert, ist das Backup der einzige Weg zur Wiederherstellung.</li>
    <li><strong>Kopplung nicht ohne Not auflösen.</strong> Werkseinstellungen, das Entfernen aus dem Midea-Konto oder ein WLAN-Modul-Tausch erzwingen eine neue Token-Beschaffung.</li>
    <li><strong>Die PortaSplit im Heimnetz abschotten.</strong> Keine Portweiterleitung, eigenes IoT-VLAN, Zugriff nur von Home Assistant.</li>
  </ol>
</aside>

Die lokale Steuerung der Midea PortaSplit beruht auf zwei gerätespezifischen Werten, Token und Key. Sie authentifizieren die Verbindung zwischen Home Assistant und Gerät und lassen sich derzeit nur über die Midea-Cloud beziehen. Daraus folgen zwei Aufgaben: die Werte so sichern, dass eine Neueinrichtung ohne Cloud möglich bleibt, und Gerät sowie Home Assistant so betreiben, dass die Werte auch bei einer Panne wenig Schaden anrichten.

Die Serie hat drei Teile: [Teil 1](/blog/midea-portasplit-home-assistant-einrichten) beschreibt die Einrichtung bis zum Dashboard, dieser Teil die Absicherung, [Teil 3](/blog/midea-v2-cloud-api-portasplit-home-assistant) die Hintergründe der Cloud-API-Warnungen.

## Woher Token und Key stammen

Bei Geräten mit dem V3-Protokoll akzeptiert die PortaSplit lokale Befehle nur mit Token und Key. Die Werte erzeugt nicht das Gerät, sondern die Midea-Cloud; auch die offizielle App bezieht sie von dort. Die Community-Integrationen haben diesen Cloud-Aufruf nachimplementiert: Sie melden sich mit denselben Endpunkten an wie die App, erhalten Token und Key und speichern beide lokal. Für den laufenden Betrieb ist danach keine Cloud-Verbindung mehr nötig.

Einen dokumentierten lokalen Pairing-Mechanismus, der die Werte ohne Cloud ausgibt, gibt es nicht. Theoretisch liessen sie sich aus der App auslesen, etwa durch Reverse Engineering oder Instrumentierung zur Laufzeit; für den einzelnen Nutzer ist das aufwändig und kein Ersatz für den Cloud-Abruf. Fällt der Endpunkt weg, fällt deshalb auch die Beschaffung weg.

Das Projekt `Midea AC LAN` warnt in seiner README, Midea schliesse die Token-Schnittstellen schrittweise; die Integration weicht deshalb von Cloud zu Cloud aus. Bereits eingerichtete Geräte laufen lokal weiter, betroffen wären neue Geräte und Neueinrichtungen. Eine verbindliche Roadmap von Midea ist das nicht. Im Juni 2026 zeigte sich zudem, dass die vermeintlich geschlossene SmartHome-Token-API weiterhin funktionierte; der Request der Community-Bibliothek war lediglich unvollständig. Die Einordnung der Warnung und der verschiedenen «V2»-Bezeichnungen steht in [Teil 3](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Was Token und Key ermöglichen

Token und Key haben keine Ablaufzeit. Laut `Midea AC LAN` galt die Client-Kommunikation ursprünglich als ausreichend geschützt, weshalb die Cloud nie ablaufende Tokens ausgab. Für sich genommen ist das keine Schwachstelle; problematisch wird es, wenn die Werte in Logs oder ungeschützten Backups landen, an Dritte gelangen oder weder widerrufen noch rotiert werden können.

Wer Token und Key besitzt und das Gerät im Netz erreicht, kann sich gegenüber der PortaSplit authentifizieren, Statusinformationen auslesen, sie ein- und ausschalten, Betriebsmodi wechseln und die Solltemperatur ändern. Die Werte allein ermöglichen keinen Angriff aus dem Internet; der Angreifer braucht zusätzlich eine Netzwerkverbindung zum Gerät. Token und Key sind deshalb wie ein Passwort zu behandeln, und das Netz sollte diese Verbindung möglichst nur Home Assistant erlauben.

Die Community-Integration greift dabei nicht das Klimagerät an. Sie implementiert ein proprietäres Protokoll, das durch Reverse Engineering nachvollzogen wurde. Das Risiko entsteht daraus, dass langlebige Geheimnisse ausserhalb der vorgesehenen App gespeichert werden.

## Token, Key und Konfiguration sichern

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

### Secrets aus einem veröffentlichten Git-Repository entfernen

Wurde eine JSON-Datei versehentlich auf GitHub veröffentlicht, reicht normales Löschen und ein neuer Commit nicht aus. Die Datei bleibt in der Git-Historie abrufbar. Mindestens diese Schritte sind nötig:

1. Repository sofort auf privat stellen, sofern möglich.
2. Datei aus der gesamten Git-Historie entfernen.
3. GitHub-Caches und Forks berücksichtigen.
4. Token als kompromittiert behandeln.
5. Gerät aus dem Midea-Konto entfernen und neu verbinden, falls dadurch neue Schlüssel erzeugt werden.
6. Home-Assistant-Integration neu einrichten.
7. Midea-Kontopasswort ändern, falls Zugangsdaten ebenfalls betroffen waren.

Ob das erneute Pairing tatsächlich einen neuen Token erzeugt, variiert je nach Gerät und Cloud-Architektur. Darauf, dass das Ändern des Kontopassworts automatisch den lokalen Gerätetoken ungültig macht, sollte man sich nicht verlassen.

## Die PortaSplit im Netz abschotten

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

## Home Assistant und Integrationen absichern

### Home Assistant als zentraler Vertrauensanker

Wer die PortaSplit lokal steuert, verlagert das Vertrauen teilweise von der Midea-Cloud zu Home Assistant. Wird Home Assistant kompromittiert, kontrolliert ein Angreifer unter Umständen nicht nur die Klimaanlage, sondern das gesamte Smart Home.

Home Assistant sollte deshalb regelmässig aktualisiert werden, nicht per ungeschützter Portweiterleitung veröffentlicht sein, mit einem starken, einzigartigen Passwort geschützt sein, Mehrfaktor-Authentifizierung verwenden, verschlüsselte Backups erstellen, nur notwendige Add-ons enthalten und keinen unnötigen SSH-Zugang aus dem Internet erlauben. Für den Fernzugriff sind ein VPN, Home Assistant Cloud oder ein sauber konfigurierter Reverse Proxy die besseren Optionen als eine simple Portweiterleitung auf Port 8123.

### HACS und das Supply-Chain-Risiko

`Midea Smart AC` und `Midea AC LAN` sind Custom Integrations. Sie laufen innerhalb von Home Assistant und erhalten damit weitreichenden Zugriff auf dessen Laufzeitumgebung. Eine bösartige oder kompromittierte Integration könnte theoretisch Konfigurationsdaten lesen, Secrets auslesen, Netzwerkverbindungen aufbauen, Geräte im lokalen Netzwerk scannen, Zustände anderer Entitäten lesen, Daten an externe Systeme übertragen und die Verfügbarkeit von Home Assistant beeinträchtigen.

Das heisst nicht, dass die genannten Integrationen bösartig sind. Beide Projekte sind öffentlich einsehbar, werden aktiv entwickelt und haben eine sichtbare Community. Open Source ist jedoch keine automatische Sicherheitsgarantie. Vor der Installation lohnt sich mindestens der Blick darauf, ob das Repository aktiv gepflegt wird, ob es regelmässige Releases gibt, wie viele Personen zum Code beitragen, ob offene Security-Issues bestehen, ob kürzlich Maintainer oder Repository-Eigentümer gewechselt haben, ob HACS auf das erwartete Repository verweist und ob ein Update ungewöhnlich grosse oder unerklärliche Änderungen enthält.

Updates sollten nicht blind unmittelbar nach Veröffentlichung installiert werden. Gerade bei sicherheitskritischen Smart-Home-Systemen ist es sinnvoll, einige Tage zu warten und Release Notes sowie gemeldete Probleme zu prüfen.

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

## Cloud und Firmware

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

### Was Midea selbst zur Sicherheit sagt

Midea wirbt für sein SmartHome-Ökosystem mit der Orientierung an mehreren Sicherheits- und Datenschutzstandards, genannt werden EN 303 645, UK PSTI, NIST, DSGVO-konforme Datenverarbeitung und die Anforderungen der EU Radio Equipment Directive. Das sind positive Signale, aber keine Aussage darüber, wie jede einzelne PortaSplit-Firmware, jeder Cloud-Endpunkt und jede lokale API tatsächlich implementiert ist. Zertifizierungs- und Marketingaussagen ersetzen keine technische Prüfung des konkreten Geräts.

Genauso wäre es falsch, aus der Warnung einer Community-Integration abzuleiten, die PortaSplit sei generell unsicher. Das beschriebene Problem betrifft die Architektur langlebiger Tokens und deren Verwendung durch inoffizielle Clients.

## Risiko nach Szenario

| Szenario | Risiko | Begründung |
| --- | --- | --- |
| Normales Heimnetz ohne Portweiterleitung | überschaubar | Ein Angreifer braucht zuerst Zugriff auf WLAN, Home Assistant oder ein Backup. |
| Flaches Heimnetz mit vielen unsicheren IoT-Geräten | mittel | Ein kompromittiertes anderes IoT-Gerät kann PortaSplit oder Home Assistant im selben Netz erreichen. |
| PortaSplit direkt aus dem Internet erreichbar | hoch | Das Gerät sollte niemals per Portweiterleitung veröffentlicht werden. |
| Token und Key öffentlich auf GitHub | hoch | Die Geheimnisse gelten als kompromittiert; ob sie widerrufen werden können, ist nicht garantiert. |
| Separates IoT-VLAN, restriktive Firewall, lokale Steuerung | vergleichsweise gering | Selbst bei einer Schwachstelle im Gerät ist die Bewegungsfreiheit im Netz stark eingeschränkt. |

## Checkliste

```text
1. Home-Assistant-Backup anfertigen
2. Token- und Konfigurationsdaten verschlüsselt sichern
3. DHCP-Reservation für die PortaSplit einrichten
4. Keine Portweiterleitung, UPnP einschränken
5. PortaSplit in ein separates IoT-VLAN verschieben
6. Zugriff von Home Assistant zur PortaSplit erlauben
7. Zugriff der PortaSplit auf interne Netze blockieren
8. Internetzugriff testweise blockieren
9. lokale Steuerung nach Neustarts prüfen
10. Firmware- und Integrationsupdates kontrolliert durchführen
```

Die gewünschte Kommunikationsrichtung:

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

So betrieben ist die lokale Steuerung unter Sicherheitsgesichtspunkten vertretbar: Token und Key bleiben geheim und gesichert, das Gerät ist nur für Home Assistant erreichbar, und Updates von Firmware und Integration werden kontrolliert eingespielt.

## Quellen

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integration `Midea AC LAN` mit der „Important Notice" (seit 19. Mai 2025, aktualisiert am 14. Juli 2025), der Begründung über nicht ablaufende Tokens und der Beschreibung des cloudbasierten Token-Bezugs.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integration `Midea Smart AC`: cloudbasierter Token- und Key-Bezug bei V3-Geräten, lokale Speicherung der Werte, Standardport 6444.

3.  [midea_ac_lan: Debug- und Konfigurationshinweise](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): Ablage der Gerätekonfiguration unter `/config/.storage/midea_ac_lan/`, Empfehlung zum Sichern statt Löschen der JSON-Datei und die Logger-Konfiguration für Debug-Logs.

4.  [Issue 779: Out Silent Mode der PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/779): Anfrage zur Unterstützung des mit dem Firmware-Update von Januar 2026 eingeführten Leisemodus des Aussengeräts, der die Geräuschentwicklung um rund 6 Dezibel senkt.

5.  [Midea SmartHome](https://www.midea.com/global/smarthome): Herstellerangaben zu den Sicherheits- und Datenschutzstandards EN 303 645, PSTI, NIST, DSGVO und RED DA.

6.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): Installation und Verwaltung von Custom Integrations, die nicht Bestandteil von Home Assistant Core sind.
