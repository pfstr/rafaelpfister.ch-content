---
title: "Midea PortaSplit in Home Assistant: Einrichtung und Dashboard"
navTitle: "PortaSplit einrichten"
description: "Schritt für Schritt von der MSmartHome-Kopplung über die Integration Midea AC LAN bis zum fertigen Dashboard mit Kennzahlen, Steuerung und Verlaufsdiagrammen."
date: "2026-07-24"
kategorie: "Home Assistant und IoT"
timeToRead: "10 Min. Lesezeit"
image: "../images/midea-portasplit-home-assistant/portasplit-dashboard-simuliert.png"
themen:
  - "smart-home-iot"
produkte:
  - "home-assistant"
protokolle:
  - "tcp"
related:
  - "midea-portasplit-home-assistant"
  - "midea-v2-cloud-api-portasplit-home-assistant"
slug: "midea-portasplit-home-assistant-einrichten"
translationId: "article-36e7710abe426781"
url: "https://rafaelpfister.ch/blog/midea-portasplit-home-assistant-einrichten"
---

Die Midea PortaSplit lässt sich mit einer Community-Integration direkt im lokalen Netz über Home Assistant steuern. In sieben Schritten entsteht von der App-Kopplung bis zum Dashboard eine lokale Steuerung mit Kennzahlen und Verlaufsdiagrammen. Dashboard, Hilfssensoren und Theme stehen im Repository <a class="gh-badge" href="https://github.com/pfstr/ha-portasplit-dashboard" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">pfstr/ha-portasplit-dashboard</span></a>.

![Home-Assistant-Dashboard der Midea PortaSplit mit simulierten Werten eines heissen Sommertags: Kennzahlen oben, Thermostat im Kühlmodus auf 22 °C, Verläufe von Raumtemperatur, Leistungsaufnahme, Tagesenergie, Kompressorfrequenz, Kompressorbetrieb und Lüfterstufe, darunter technische Werte und Status.](../images/midea-portasplit-home-assistant/portasplit-dashboard-simuliert.png)

Die Abbildung zeigt das fertige Dashboard mit simulierten Werten: Die PortaSplit kühlt ab 17 Uhr einen auf 27 °C aufgeheizten Raum auf 22 °C, ist nachts ausgeschaltet und regelt ab 11 Uhr mit dem Inverter-Kompressor gegen die Mittagshitze.

Die Serie hat drei Teile: Teil 1 beschreibt die Einrichtung, [Teil 2](/blog/midea-portasplit-home-assistant) behandelt die Absicherung von Token, Key und Heimnetz, [Teil 3](/blog/midea-v2-cloud-api-portasplit-home-assistant) ordnet die Warnungen zur Midea-Cloud-API ein.

## Wie die lokale Steuerung funktioniert

Steuerbefehle gehen nach der Einrichtung direkt von Home Assistant an die PortaSplit, ohne Umweg über einen Midea-Server. Bei Geräten mit dem V3-Protokoll akzeptiert die PortaSplit lokale Befehle aber nur mit zwei gerätespezifischen Werten, Token und Key. Die Integration holt beide einmalig bei der Einrichtung über die Midea-Cloud ab und speichert sie lokal:

```text
Einrichtung:   Home Assistant → Midea-Cloud → Token und Key
Betrieb:       Home Assistant → lokales Netz (6444/TCP) → PortaSplit
```

Die beschriebenen Integrationen stammen aus der Community und werden weder von Midea noch von Home Assistant offiziell unterstützt. Firmware- oder Cloud-Änderungen können ihr Verhalten beeinflussen.

## Welche Integration passt

Zwei Community-Integrationen unterstützen die PortaSplit:

| Integration | Schwerpunkt |
|---|---|
| <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> (`Midea AC LAN`) | viele Midea-Geräteklassen; stellt bei der PortaSplit 21 Sensoren bereit, darunter Kompressorfrequenz, -strom und -spannung sowie Verdampfer-, Kondensator- und Heissgastemperatur |
| <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> (`Midea Smart AC`) | auf Klimageräte (`0xAC`, `0xCC`) zugeschnitten, fragt die Gerätefähigkeiten ab und unterstützt den Leisemodus des Aussengeräts |

Für meine PortaSplit verwende ich `Midea AC LAN`; Anleitung und Dashboard bauen auf deren Entitäten auf. Mit `Midea Smart AC` heissen die Entitäten anders, das Dashboard lässt sich dann nur nach Anpassung verwenden. Beide Integrationen gleichzeitig mit demselben Gerät zu betreiben führt zu Statusproblemen und ist nicht sinnvoll.

## Voraussetzungen

- Midea PortaSplit mit WLAN-Funktion und ein 2,4-GHz-WLAN
- MSmartHome-App mit Midea-Konto
- Home Assistant ab Version 2024.10 (getestet mit 2026.7) mit Zugriff auf das Konfigurationsverzeichnis `/config`, etwa über das Add-on File editor, Samba oder SSH
- HACS für die Diagrammkarte `apexcharts-card`
- Netzwerkzugriff von Home Assistant zur PortaSplit auf Port 6444/TCP

## Schritt 1: PortaSplit mit MSmartHome verbinden

1. MSmartHome-App installieren und mit dem Midea-Konto anmelden.
2. PortaSplit in den WLAN-Kopplungsmodus versetzen und mit dem 2,4-GHz-WLAN verbinden.
3. Prüfen, ob sich die PortaSplit über die App steuern lässt.
4. Im Router eine DHCP-Reservation für die PortaSplit anlegen, damit sie dauerhaft dieselbe IP-Adresse erhält.

Nutzt der Router dieselbe SSID für 2,4 und 5 GHz, funktioniert die Kopplung meist trotzdem. Bei Problemen hilft vorübergehend ein separates 2,4-GHz-WLAN.

## Schritt 2: Midea AC LAN installieren

**Über HACS:** HACS öffnen, nach `Midea AC LAN` suchen, die Integration herunterladen und Home Assistant neu starten.

**Ohne HACS:** Laden Sie das Release-Archiv direkt in das Verzeichnis `custom_components`. In einer Docker-Installation geht das mit `docker exec -it homeassistant bash` im Container, beim Home Assistant OS im Terminal-Add-on:

```bash
mkdir -p /config/custom_components
cd /config/custom_components
wget https://github.com/wuwentao/midea_ac_lan/releases/download/v2026.9.2/midea_ac_lan.zip
unzip midea_ac_lan.zip -d midea_ac_lan
rm midea_ac_lan.zip
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `mkdir -p` | legt das Verzeichnis an, falls es fehlt, und meldet keinen Fehler, wenn es schon existiert |
| `wget <url>` | lädt das Release-Archiv der angegebenen Version von GitHub herunter |
| `unzip <archiv>` | entpackt das Archiv |
| `-d midea_ac_lan` | Zielverzeichnis; die Dateien liegen im Archiv ohne Unterordner und müssen in `custom_components/midea_ac_lan/` landen |

</details>

Die aktuelle Versionsnummer steht auf der Release-Seite des Projekts. Starten Sie Home Assistant danach über Einstellungen, System und Neu starten. Die Warnung `We found a custom integration midea_ac_lan which has not been tested by Home Assistant` im Protokoll ist bei jeder Custom Integration normal.

## Schritt 3: PortaSplit hinzufügen

Unter Einstellungen, Geräte & Dienste, Integration hinzufügen nach `Midea AC LAN` suchen. Der Einrichtungsdialog fragt nacheinander:

1. **Aktion:** `Discover automatically`.
2. **IP-Adresse:** `auto` durchsucht das lokale Netz. Liegt die PortaSplit in einem anderen VLAN, tragen Sie ihre IP-Adresse ein, weil die Suche per Broadcast nicht über VLAN-Grenzen reicht.
3. **Gerät:** Die PortaSplit erscheint als `<Geräte-ID> (Air Conditioner)`.
4. **Login:** Konto, Passwort und Server. Für ein Konto der MSmartHome-App wählen Sie `SmartHome`; scheitert die Anmeldung, `NetHome Plus` mit denselben Zugangsdaten. Die Integration holt damit einmalig Token und Key.

Danach erscheint das Gerät mit einer einzigen Entität `climate.<geräte-id>_climate`. Die Geräte-ID ist eine 15-stellige Zahl und wird im Folgenden als `DEVICE_ID` bezeichnet.

## Schritt 4: Sensoren freischalten

`Midea AC LAN` legt standardmässig nur die Klima-Entität an. Unter Einstellungen, Geräte & Dienste, Midea AC LAN, Konfigurieren öffnet sich der Optionsdialog:

| Feld | Einstellung |
|---|---|
| IP-Adresse | unverändert lassen |
| Refresh interval | 30 Sekunden (Standard) |
| Sensors | alle angebotenen Sensoren auswählen |
| Switches | mindestens `Power`, `ECO Mode`, `Sleep Mode`, `Swing Vertical`, `Swing Horizontal`, `Screen Display`, `Prompt Tone` und `Fan Speed Percent` |
| Customize | leer lassen |

Nach dem Speichern legt die Integration die Entitäten ohne Neustart an, nach dem Muster `sensor.DEVICE_ID_indoor_temperature`. Eine PortaSplit (Gerätetyp `0xAC`, Protokoll V3) lieferte mit `Midea AC LAN` v2026.9.2 im Standby diese Werte:

| Entität | Bedeutung | Wert im Standby |
|---|---|---|
| `indoor_temperature` | Raumtemperatur | 23,0 °C |
| `outdoor_temperature` | Aussentemperatur am Aussengerät | 23,5 °C |
| `realtime_power` | aktuelle Leistungsaufnahme | 1,5 W |
| `total_energy_consumption` | Energiezähler seit Inbetriebnahme | 90,54 kWh |
| `compressor_frequency`, `target_compressor_frequency` | Kompressorfrequenz Ist und Soll | 0 Hz |
| `compressor_voltage`, `compressor_current`, `compressor_power` | Spannung, Strom und Leistung am Kompressor | 230 V, 1 A, 4 W |
| `indoor_coil_temperature` (T2), `outdoor_coil_temperature` (T3) | Verdampfer, Kondensator | 23,5 °C |
| `discharge_pipe_temperature` (TP) | Heissgasleitung | 23 °C |
| `indoor_fan_speed` | Lüfterdrehzahl | 0 rpm |
| `error_code`, `full_dust` | Fehlercode, Filter verschmutzt | 0, on |
| `indoor_humidity` | Luftfeuchtigkeit | unknown |

Die PortaSplit besitzt keinen Feuchtesensor, `indoor_humidity` bleibt deshalb leer. Die Klima-Entität kennt die Modi Aus, Auto, Kühlen, Entfeuchten, Heizen und Lüften, Solltemperaturen von 16 bis 30 °C in Schritten von 0,5 °C sowie die Lüfterstufen Silent, Low, Medium, High, Full und Auto.

## Schritt 5: Diagrammkarte installieren

Die Verlaufsdiagramme verwenden `apexcharts-card`. In HACS nach `apexcharts-card` suchen und herunterladen. HACS registriert die Karte als Dashboard-Ressource; danach den Browser einmal neu laden.

## Schritt 6: Hilfssensoren und Theme einrichten

Das Dashboard benötigt fünf Hilfssensoren: Kompressor an/aus, Lüfterstufe als Zahl für das Stufendiagramm, Laufzeit und Energie seit Mitternacht sowie den Zeitpunkt der letzten Meldung. Aktivieren Sie zuerst Pakete und Themes in der `configuration.yaml`, sofern noch nicht geschehen:

```yaml
homeassistant:
  packages: !include_dir_named packages

frontend:
  themes: !include_dir_merge_named themes
```

Kopieren Sie dann aus dem Repository `packages/portasplit.yaml` nach `/config/packages/` und `themes/portasplit.yaml` nach `/config/themes/` und ersetzen Sie in der Paketdatei jedes `DEVICE_ID` durch die eigene Geräte-ID. Das Paket enthält:

```yaml
template:
  - binary_sensor:
      - name: "PortaSplit Kompressor"
        unique_id: portasplit_kompressor
        icon: mdi:heat-pump-outline
        device_class: running
        state: >
          {% set f = states('sensor.DEVICE_ID_compressor_frequency') %}
          {% if f | is_number %}{{ f | float > 0 }}
          {% else %}{{ states('sensor.DEVICE_ID_realtime_power') | float(0) > 150 }}{% endif %}
  - sensor:
      - name: "PortaSplit Lüfterstufe"
        unique_id: portasplit_luefterstufe_num
        icon: mdi:fan
        state: >
          {% set m = state_attr('climate.DEVICE_ID_climate', 'fan_mode') %}
          {{ {'silent': 1, 'low': 2, 'medium': 3, 'high': 4,
              'full': 5, 'auto': 6}.get(m, none) }}
      - name: "PortaSplit letzte Meldung"
        unique_id: portasplit_letzte_meldung
        icon: mdi:sync
        device_class: timestamp
        state: "{{ states.climate['DEVICE_ID_climate'].last_reported }}"

sensor:
  - platform: history_stats
    name: "PortaSplit Laufzeit heute"
    unique_id: portasplit_laufzeit_heute
    entity_id: binary_sensor.portasplit_kompressor
    state: "on"
    type: time
    start: "{{ today_at() }}"
    end: "{{ now() }}"

utility_meter:
  portasplit_energie_heute:
    name: "PortaSplit Energie heute"
    unique_id: portasplit_energie_heute
    source: sensor.DEVICE_ID_total_energy_consumption
    cycle: daily
```

<details class="options-details">
<summary>Optionen erklärt</summary>

| Option | Wirkung |
|---|---|
| `template: binary_sensor` | Kompressor gilt als laufend, wenn die Kompressorfrequenz über 0 Hz liegt; meldet ein Gerät keine Frequenz, gilt eine Leistung über 150 W als Kriterium |
| `template: sensor` (Lüfterstufe) | übersetzt den Lüftermodus in eine Zahl von 1 (Silent) bis 6 (Auto), damit das Diagramm Stufen zeichnen kann |
| `states.climate['…']` | Klammerschreibweise ist nötig, weil die Objekt-ID mit Ziffern beginnt; `states.climate.123…` ist für Jinja ungültig |
| `last_reported` | Zeitpunkt der letzten Meldung des Geräts, auch wenn sich kein Wert geändert hat |
| `history_stats` mit `type: time` | summiert die Zeit, in der der Kompressor-Sensor `on` war |
| `start` / `end` | Zeitraum von Mitternacht bis jetzt |
| `utility_meter` mit `cycle: daily` | bildet aus dem Gesamtzähler einen Tageszähler, der um Mitternacht auf 0 zurückgesetzt wird |

</details>

Prüfen Sie die Konfiguration unter Entwicklerwerkzeuge, YAML, Konfiguration prüfen und starten Sie Home Assistant neu; `utility_meter` lässt sich nicht per Reload laden. Danach existieren `binary_sensor.portasplit_kompressor`, `sensor.portasplit_lufterstufe`, `sensor.portasplit_laufzeit_heute`, `sensor.portasplit_energie_heute` und `sensor.portasplit_letzte_meldung`. Home Assistant lässt beim Bilden der Entitäts-ID den Umlaut weg, daher `lufterstufe`.

## Schritt 7: Dashboard anlegen

1. Einstellungen, Dashboards, Dashboard hinzufügen, Neues Dashboard von Grund auf, Name `PortaSplit`.
2. Das neue Dashboard öffnen und über den Stift in den Bearbeitungsmodus wechseln.
3. Über das Drei-Punkte-Menü den Raw-Konfigurationseditor öffnen.
4. Den Inhalt vollständig durch `dashboard.yaml` aus dem Repository ersetzen, vorher jedes `DEVICE_ID` durch die eigene Geräte-ID ersetzen.
5. Speichern und den Bearbeitungsmodus schliessen.

Das Dashboard verwendet die Ansicht «Abschnitte» mit vier Spalten und das Theme `PortaSplit Dark`. Oben stehen acht Kennzahlen, links die Steuerung mit Thermostat, Betriebsmodus und Lüfterstufe, rechts die Verläufe der letzten 24 Stunden. Darunter folgen die technischen Werte des Kältekreislaufs und ein Statusblock mit Fehlercode, Filterzustand und letzter Meldung.

Direkt nach der Einrichtung sind die Diagramme leer und füllen sich mit der Laufzeit. `Energie heute` zeigt `Unbekannt`, bis der Energiezähler zum ersten Mal steigt.

## Nach der Einrichtung

Token und Key liegen jetzt in Home Assistant. Lassen sie sich später nicht mehr über die Cloud beziehen, ist ein Backup der einzige Weg zu einer Neueinrichtung. Wie Sie Token, Key und Konfiguration sichern und die PortaSplit im Heimnetz abschotten, beschreibt [Teil 2: PortaSplit absichern](/blog/midea-portasplit-home-assistant).

## Quellen

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integration `Midea AC LAN`: unterstützte Geräteklassen, Installation über HACS, Mindestversion Home Assistant 2024.4.1.

2.  [midea_ac_lan: Releases](https://github.com/wuwentao/midea_ac_lan/releases): Release-Archive für die Installation ohne HACS, getestet mit v2026.9.2.

3.  [midea_ac_lan: Dokumentation der Klimaentitäten](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md): Entitäten und Attribute für Klimageräte, darunter Leistung, Gesamtenergie und Kompressorfrequenz.

4.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integration `Midea Smart AC`: unterstützte Gerätetypen `0xAC` und `0xCC`, PortaSplit mit „Out Silent Mode", Cloud-Nutzung zur Token- und Key-Beschaffung bei V3-Geräten und Standardport 6444.

5.  <a class="gh-badge" href="https://github.com/pfstr/ha-portasplit-dashboard" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">pfstr/ha-portasplit-dashboard</span></a>: Dashboard, Hilfssensoren-Paket und Theme aus dieser Anleitung, mit der Liste der Entitäten, die eine PortaSplit mit `Midea AC LAN` v2026.9.2 meldet.

6.  [apexcharts-card](https://github.com/RomRider/apexcharts-card): Diagrammkarte für die Verlaufsdiagramme.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): Installation von Custom Integrations und Frontend-Karten.

8.  [Home Assistant: Packages](https://www.home-assistant.io/docs/configuration/packages/): Bündeln von Template-, Sensor- und Utility-Meter-Konfiguration in einer Datei unter `/config/packages/`.

9.  [Home Assistant: History Stats](https://www.home-assistant.io/integrations/history_stats/): Sensor-Plattform für die Laufzeit des Kompressors seit Mitternacht.

10.  [Home Assistant: Utility Meter](https://www.home-assistant.io/integrations/utility_meter/): Tageszähler auf Basis des Gesamtenergiezählers.
