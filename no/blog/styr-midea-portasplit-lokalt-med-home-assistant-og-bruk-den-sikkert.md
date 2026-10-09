---
title: "Midea PortaSplit i Home Assistant: Oppsett og dashbord"
navTitle: "Sett opp PortaSplit"
description: "Trinn for trinn fra MSmartHome-tilkobling via integrasjonen Midea AC LAN til et ferdig dashbord med nøkkeltall, styring og historikkdiagrammer."
date: "2026-07-24"
kategorie: "Home Assistant og IoT"
timeToRead: "10 min. lesetid"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant-absichern
  - midea-v2-cloud-api-portasplit-home-assistant
slug: "styr-midea-portasplit-lokalt-med-home-assistant-og-bruk-den-sikkert"
translationOf: "midea-portasplit-home-assistant"
translationId: article-36e7710abe426781
translationReview: automatic
translationSourceHash: 6b0bf224030d5fca539c146523bb8de015a6bab426c4e35ab116423719d9c232
translatedAt: 2026-10-09T10:55:40.034Z
translationModel: gpt-5.6-terra
image: ../images/midea-portasplit-home-assistant/portasplit-dashboard.png
url: https://rafaelpfister.ch/no/blog/styr-midea-portasplit-lokalt-med-home-assistant-og-bruk-den-sikkert
---

Midea PortaSplit kan styres direkte på det lokale nettverket via Home Assistant med en community-integrasjon. I sju trinn får du lokal styring med nøkkeltall og historikkdiagrammer, fra app-tilkobling til dashbord. Dashbord, hjelpesensorer og tema finnes i repositoriet <a class="gh-badge" href="https://github.com/pfstr/ha-portasplit-dashboard" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">pfstr/ha-portasplit-dashboard</span></a>.

![Home Assistant-dashbord for Midea PortaSplit i kjøledrift: nøkkeltall øverst, termostat på 22 °C, historikk for romtemperatur, effektforbruk, dagsenergi, kompressorfrekvens, kompressordrift og viftehastighet, med tekniske verdier og status nedenfor.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

Illustrasjonen viser det ferdige dashbordet i kjøledrift med nøkkeltall, styring og historikken for de siste 24 timene.

Serien består av tre deler: Del 1 beskriver oppsettet, [del 2](/blog/midea-portasplit-home-assistant-absichern) omhandler sikring av token, key og hjemmenettverk, og [del 3](/blog/midea-v2-cloud-api-portasplit-home-assistant) setter advarslene om Midea Cloud API i sammenheng.

## Slik fungerer lokal styring

Etter oppsettet sendes styringskommandoer direkte fra Home Assistant til PortaSplit, uten omvei via en Midea-server. På enheter med V3-protokollen godtar PortaSplit imidlertid bare lokale kommandoer med to enhetsspesifikke verdier: token og key. Integrasjonen henter begge én gang under oppsettet fra Midea Cloud og lagrer dem lokalt:

```text
Einrichtung:   Home Assistant → Midea-Cloud → Token und Key
Betrieb:       Home Assistant → lokales Netz (6444/TCP) → PortaSplit
```

Integrasjonene som beskrives, kommer fra community-et og støttes ikke offisielt av verken Midea eller Home Assistant. Endringer i fastvare eller skyen kan påvirke hvordan de fungerer.

## Hvilken integrasjon passer

To community-integrasjoner støtter PortaSplit:

| Integrasjon | Hovedfokus |
|---|---|
| <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> (`Midea AC LAN`) | mange Midea-enhetsklasser; gir 21 sensorer for PortaSplit, blant annet kompressorfrekvens, -strøm og -spenning samt fordamper-, kondensator- og hetgasstemperatur |
| <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> (`Midea Smart AC`) | tilpasset klimaanlegg (`0xAC`, `0xCC`), henter enhetsegenskapene og støtter stillemodus for utedelen |

For min PortaSplit bruker jeg `Midea AC LAN`; veiledningen og dashbordet bygger på entitetene derfra. Med `Midea Smart AC` har entitetene andre navn, og dashbordet kan da bare brukes etter tilpasning. Å bruke begge integrasjonene samtidig med samme enhet fører til statusproblemer og er ikke fornuftig.

## Forutsetninger

- Midea PortaSplit med WLAN-funksjon og et 2,4 GHz-WLAN
- MSmartHome-app med Midea-konto
- Home Assistant fra versjon 2024.10 (testet med 2026.7) med tilgang til konfigurasjonsmappen `/config`, for eksempel via tillegget File editor, Samba eller SSH
- HACS for diagramkortet `apexcharts-card`
- Nettverkstilgang fra Home Assistant til PortaSplit på port 6444/TCP

## Trinn 1: Koble PortaSplit til MSmartHome

1. Installer MSmartHome-appen og logg inn med Midea-kontoen.
2. Sett PortaSplit i WLAN-tilkoblingsmodus og koble den til 2,4 GHz-WLAN-et.
3. Kontroller at PortaSplit kan styres via appen.
4. Opprett en DHCP-reservasjon for PortaSplit i ruteren, slik at den alltid får samme IP-adresse.

Hvis ruteren bruker samme SSID for 2,4 og 5 GHz, fungerer tilkoblingen som regel likevel. Ved problemer kan et separat 2,4 GHz-WLAN hjelpe midlertidig.

## Trinn 2: Installer Midea AC LAN

**Via HACS:** Åpne HACS, søk etter `Midea AC LAN`, last ned integrasjonen og start Home Assistant på nytt.

**Uten HACS:** Last ned utgivelsesarkivet direkte til mappen `custom_components`. I en Docker-installasjon kan dette gjøres med `docker exec -it homeassistant bash` i containeren, og i Home Assistant OS med terminaltillegget:

```bash
mkdir -p /config/custom_components
cd /config/custom_components
wget https://github.com/wuwentao/midea_ac_lan/releases/download/v2026.9.2/midea_ac_lan.zip
unzip midea_ac_lan.zip -d midea_ac_lan
rm midea_ac_lan.zip
```

<details class="options-details">
<summary>Alternativene forklart</summary>

| Alternativ | Virkning |
|---|---|
| `mkdir -p` | oppretter mappen dersom den mangler, og gir ingen feil hvis den allerede finnes |
| `wget <url>` | laster ned utgivelsesarkivet for den angitte versjonen fra GitHub |
| `unzip <archiv>` | pakker ut arkivet |
| `-d midea_ac_lan` | målmappen; filene ligger uten undermappe i arkivet og må havne i `custom_components/midea_ac_lan/` |

</details>

Det gjeldende versjonsnummeret finner du på prosjektets utgivelsesside. Start deretter Home Assistant på nytt via Innstillinger, System og Start på nytt. Advarselen `We found a custom integration midea_ac_lan which has not been tested by Home Assistant` i loggen er normal for enhver Custom Integration.

## Trinn 3: Legg til PortaSplit

Gå til Innstillinger, Enheter og tjenester, Legg til integrasjon, og søk etter `Midea AC LAN`. Oppsettsdialogen spør etter følgende i rekkefølge:

1. **Handling:** `Discover automatically`.
2. **IP-adresse:** `auto` søker gjennom det lokale nettverket. Hvis PortaSplit er i et annet VLAN, oppgir du IP-adressen, siden søket med broadcast ikke krysser VLAN-grenser.
3. **Enhet:** PortaSplit vises som `<Geräte-ID> (Air Conditioner)`.
4. **Innlogging:** Konto, passord og server. For en konto i MSmartHome-appen velger du `SmartHome`; mislykkes innloggingen, velger du `NetHome Plus` med de samme påloggingsopplysningene. Integrasjonen henter dermed token og key én gang.

Deretter vises enheten med én enkelt entitet `climate.<geräte-id>_climate`. Enhets-ID-en er et 15-sifret tall og omtales videre som `DEVICE_ID`.

## Trinn 4: Aktiver sensorer

`Midea AC LAN` oppretter som standard bare klimaentiteten. Gå til Innstillinger, Enheter og tjenester, Midea AC LAN, Konfigurer for å åpne alternativdialogen:

| Felt | Innstilling |
|---|---|
| IP-adresse | la stå uendret |
| Refresh interval | 30 sekunder (standard) |
| Sensors | velg alle tilgjengelige sensorer |
| Switches | minst `Power`, `ECO Mode`, `Sleep Mode`, `Swing Vertical`, `Swing Horizontal`, `Screen Display`, `Prompt Tone` og `Fan Speed Percent` |
| Customize | la stå tomt |

Etter lagring oppretter integrasjonen entitetene uten omstart, etter mønsteret `sensor.DEVICE_ID_indoor_temperature`. En PortaSplit (enhetstype `0xAC`, protokoll V3) leverte disse verdiene i standby med `Midea AC LAN` v2026.9.2:

| Entitet | Betydning | Verdi i standby |
|---|---|---|
| `indoor_temperature` | romtemperatur | 23,0 °C |
| `outdoor_temperature` | utetemperatur ved utedelen | 23,5 °C |
| `realtime_power` | aktuelt effektforbruk | 1,5 W |
| `total_energy_consumption` | energimåler siden oppstart | 90,54 kWh |
| `compressor_frequency`, `target_compressor_frequency` | faktisk og ønsket kompressorfrekvens | 0 Hz |
| `compressor_voltage`, `compressor_current`, `compressor_power` | spenning, strøm og effekt ved kompressoren | 230 V, 1 A, 4 W |
| `indoor_coil_temperature` (T2), `outdoor_coil_temperature` (T3) | fordamper, kondensator | 23,5 °C |
| `discharge_pipe_temperature` (TP) | hetgassledning | 23 °C |
| `indoor_fan_speed` | viftehastighet | 0 rpm |
| `error_code`, `full_dust` | feilkode, skittent filter | 0, on |
| `indoor_humidity` | luftfuktighet | unknown |

PortaSplit har ingen fuktighetssensor, så `indoor_humidity` forblir tom. Klimaentiteten kjenner modusene Av, Auto, Kjøling, Avfukting, Oppvarming og Vifte, ønsketemperaturer fra 16 til 30 °C i trinn på 0,5 °C, samt viftehastighetene Silent, Low, Medium, High, Full og Auto.

## Trinn 5: Installer diagramkortet

Historikkdiagrammene bruker `apexcharts-card`. Søk etter `apexcharts-card` i HACS og last det ned. HACS registrerer kortet som en dashbordressurs; last deretter nettleseren på nytt én gang.

## Trinn 6: Sett opp hjelpesensorer og tema

Dashbordet trenger fem hjelpesensorer: kompressor av/på, viftehastighet som tall for trinnediagrammet, driftstid og energi siden midnatt samt tidspunktet for siste melding. Aktiver først pakker og temaer i `configuration.yaml`, dersom dette ikke allerede er gjort:

```yaml
homeassistant:
  packages: !include_dir_named packages

frontend:
  themes: !include_dir_merge_named themes
```

Kopier deretter `packages/portasplit.yaml` fra repositoriet til `/config/packages/` og `themes/portasplit.yaml` til `/config/themes/`, og erstatt hver forekomst av `DEVICE_ID` i pakkefilen med din egen enhets-ID. Pakken inneholder:

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
<summary>Alternativene forklart</summary>

| Alternativ | Virkning |
|---|---|
| `template: binary_sensor` | kompressoren regnes som i drift når kompressorfrekvensen er over 0 Hz; hvis en enhet ikke rapporterer frekvens, brukes effekt over 150 W som kriterium |
| `template: sensor` (viftehastighet) | oversetter viftemodus til et tall fra 1 (Silent) til 6 (Auto), slik at diagrammet kan tegne trinn |
| `states.climate['…']` | parentesnotasjon er nødvendig fordi objekt-ID-en begynner med et siffer; `states.climate.123…` er ugyldig for Jinja |
| `last_reported` | tidspunktet for enhetens siste melding, også når ingen verdi har endret seg |
| `history_stats` med `type: time` | summerer tiden da kompressorsensoren var `on` |
| `start` / `end` | tidsrommet fra midnatt til nå |
| `utility_meter` med `cycle: daily` | oppretter en dagsmåler fra totalmåleren, som tilbakestilles til 0 ved midnatt |

</details>

Kontroller konfigurasjonen under Utviklerverktøy, YAML, Kontroller konfigurasjon og start Home Assistant på nytt; `utility_meter` kan ikke lastes inn med Reload. Deretter finnes `binary_sensor.portasplit_kompressor`, `sensor.portasplit_lufterstufe`, `sensor.portasplit_laufzeit_heute`, `sensor.portasplit_energie_heute` og `sensor.portasplit_letzte_meldung`. Home Assistant utelater omlauten når entitets-ID-en dannes, derfor `lufterstufe`.

## Trinn 7: Opprett dashbordet

1. Gå til Innstillinger, Dashbord, Legg til dashbord, Nytt dashbord fra grunnen av, navn `PortaSplit`.
2. Åpne det nye dashbordet og gå til redigeringsmodus via blyantikonet.
3. Åpne Raw-konfigurasjonseditoren via menyen med tre prikker.
4. Erstatt hele innholdet med `dashboard.yaml` fra repositoriet, og erstatt først hver forekomst av `DEVICE_ID` med din egen enhets-ID.
5. Lagre og lukk redigeringsmodusen.

Dashbordet bruker visningen «Seksjoner» med fire kolonner og temaet `PortaSplit Dark`. Øverst vises åtte nøkkeltall, til venstre styringen med termostat, driftsmodus og viftehastighet, og til høyre historikken for de siste 24 timene. Under følger de tekniske verdiene for kjølekretsen og en statusblokk med feilkode, filterstatus og siste melding.

Rett etter oppsettet er diagrammene tomme og fylles opp over tid. `Energie heute` viser `Unbekannt` til energimåleren øker for første gang.

## Etter oppsettet

Token og key ligger nå i Home Assistant. Hvis de senere ikke lenger kan hentes fra skyen, er en sikkerhetskopi den eneste måten å sette opp på nytt. [Del 2: Sikre PortaSplit](/blog/midea-portasplit-home-assistant-absichern) beskriver hvordan du sikrer token, key og konfigurasjon og isolerer PortaSplit i hjemmenettverket.

## Kilder

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integrasjonen `Midea AC LAN`: støttede enhetsklasser, installasjon via HACS, minimumsversjon Home Assistant 2024.4.1.

2.  [midea_ac_lan: Utgivelser](https://github.com/wuwentao/midea_ac_lan/releases): Utgivelsesarkiver for installasjon uten HACS, testet med v2026.9.2.

3.  [midea_ac_lan: Dokumentasjon for klimaentiteter](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md): Entiteter og attributter for klimaanlegg, blant annet effekt, totalenergi og kompressorfrekvens.

4.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integrasjonen `Midea Smart AC`: støttede enhetstyper `0xAC` og `0xCC`, PortaSplit med «Out Silent Mode», skybruk for å hente token og key for V3-enheter samt standardport 6444.

5.  <a class="gh-badge" href="https://github.com/pfstr/ha-portasplit-dashboard" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">pfstr/ha-portasplit-dashboard</span></a>: Dashbord, hjelpesensorpakke og tema fra denne veiledningen, med listen over entitetene som en PortaSplit med `Midea AC LAN` v2026.9.2 rapporterer.

6.  [apexcharts-card](https://github.com/RomRider/apexcharts-card): Diagramkort for historikkdiagrammene.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): Installasjon av Custom Integrations og frontend-kort.

8.  [Home Assistant: Packages](https://www.home-assistant.io/docs/configuration/packages/): Samling av Template-, Sensor- og Utility Meter-konfigurasjon i én fil under `/config/packages/`.

9.  [Home Assistant: History Stats](https://www.home-assistant.io/integrations/history_stats/): Sensorplattform for kompressorens driftstid siden midnatt.

10.  [Home Assistant: Utility Meter](https://www.home-assistant.io/integrations/utility_meter/): Dagsmåler basert på totalenergimåleren.
