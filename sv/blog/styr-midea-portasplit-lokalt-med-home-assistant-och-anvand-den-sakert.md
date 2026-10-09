---
title: "Midea PortaSplit i Home Assistant: installation och dashboard"
navTitle: "Installera PortaSplit"
description: "Steg för steg från parkoppling med MSmartHome via integrationen Midea AC LAN till ett färdigt dashboard med nyckeltal, styrning och historikdiagram."
date: "2026-07-24"
kategorie: "Home Assistant och IoT"
timeToRead: "10 min lästid"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant-absichern
  - midea-v2-cloud-api-portasplit-home-assistant
slug: "styr-midea-portasplit-lokalt-med-home-assistant-och-anvand-den-sakert"
translationOf: "midea-portasplit-home-assistant"
translationId: article-36e7710abe426781
translationReview: automatic
translationSourceHash: 6b0bf224030d5fca539c146523bb8de015a6bab426c4e35ab116423719d9c232
translatedAt: 2026-10-09T10:54:23.503Z
translationModel: gpt-5.6-terra
image: ../images/midea-portasplit-home-assistant/portasplit-dashboard.png
url: https://rafaelpfister.ch/sv/blog/styr-midea-portasplit-lokalt-med-home-assistant-och-anvand-den-sakert
---

Midea PortaSplit kan styras direkt i det lokala nätverket via Home Assistant med en community-integration. I sju steg skapas lokal styrning med nyckeltal och historikdiagram, från parkoppling med appen till dashboardet. Dashboard, hjälpsensorer och tema finns i repositoryt <a class="gh-badge" href="https://github.com/pfstr/ha-portasplit-dashboard" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">pfstr/ha-portasplit-dashboard</span></a>.

![Home Assistant-dashboard för Midea PortaSplit i kylläge: nyckeltal högst upp, termostat inställd på 22 °C, historik för rumstemperatur, effektförbrukning, daglig energi, kompressorfrekvens, kompressordrift och fläkthastighet, följt av tekniska värden och status.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

Bilden visar det färdiga dashboardet i kylläge med nyckeltal, styrning och historik för de senaste 24 timmarna.

Serien består av tre delar: Del 1 beskriver installationen, [del 2](/blog/midea-portasplit-home-assistant-absichern) handlar om att säkra token, key och hemnätverket, [del 3](/blog/midea-v2-cloud-api-portasplit-home-assistant) sätter varningarna om Midea Cloud API i sitt sammanhang.

## Så fungerar den lokala styrningen

Efter installationen går styrkommandon direkt från Home Assistant till PortaSplit, utan omväg via en Midea-server. På enheter med V3-protokollet accepterar PortaSplit dock lokala kommandon endast med två enhetsspecifika värden, token och key. Integrationen hämtar båda en gång vid installationen via Midea Cloud och lagrar dem lokalt:

```text
Einrichtung:   Home Assistant → Midea-Cloud → Token und Key
Betrieb:       Home Assistant → lokales Netz (6444/TCP) → PortaSplit
```

De beskrivna integrationerna kommer från communityn och stöds inte officiellt av vare sig Midea eller Home Assistant. Firmware- eller molnändringar kan påverka deras funktion.

## Vilken integration passar

Två community-integrationer stöder PortaSplit:

| Integration | Fokus |
|---|---|
| <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> (`Midea AC LAN`) | många Midea-enhetsklasser; tillhandahåller 21 sensorer för PortaSplit, bland annat kompressorfrekvens, -ström och -spänning samt förångar-, kondensor- och hetgastemperatur |
| <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> (`Midea Smart AC`) | anpassad för luftkonditioneringsenheter (`0xAC`, `0xCC`), läser av enhetsfunktioner och stöder utomhusenhetens tystläge |

För min PortaSplit använder jag `Midea AC LAN`; instruktionerna och dashboardet bygger på dess entiteter. Med `Midea Smart AC` har entiteterna andra namn, och dashboardet kan då bara användas efter anpassning. Att köra båda integrationerna samtidigt med samma enhet leder till statusproblem och är inte meningsfullt.

## Förutsättningar

- Midea PortaSplit med Wi-Fi-funktion och ett 2,4 GHz-Wi-Fi
- MSmartHome-appen med Midea-konto
- Home Assistant från version 2024.10 (testat med 2026.7) med åtkomst till konfigurationskatalogen `/config`, till exempel via tillägget File editor, Samba eller SSH
- HACS för diagramkortet `apexcharts-card`
- Nätverksåtkomst från Home Assistant till PortaSplit på port 6444/TCP

## Steg 1: Anslut PortaSplit till MSmartHome

1. Installera MSmartHome-appen och logga in med Midea-kontot.
2. Sätt PortaSplit i Wi-Fi-parkopplingsläge och anslut den till 2,4 GHz-Wi-Fi.
3. Kontrollera att PortaSplit kan styras via appen.
4. Skapa en DHCP-reservation för PortaSplit i routern så att den permanent får samma IP-adress.

Om routern använder samma SSID för 2,4 och 5 GHz fungerar parkopplingen oftast ändå. Vid problem hjälper det att tillfälligt använda ett separat 2,4 GHz-Wi-Fi.

## Steg 2: Installera Midea AC LAN

**Via HACS:** Öppna HACS, sök efter `Midea AC LAN`, hämta integrationen och starta om Home Assistant.

**Utan HACS:** Ladda release-arkivet direkt till katalogen `custom_components`. I en Docker-installation kan detta göras med `docker exec -it homeassistant bash` i containern, och i Home Assistant OS med terminaltillägget:

```bash
mkdir -p /config/custom_components
cd /config/custom_components
wget https://github.com/wuwentao/midea_ac_lan/releases/download/v2026.9.2/midea_ac_lan.zip
unzip midea_ac_lan.zip -d midea_ac_lan
rm midea_ac_lan.zip
```

<details class="options-details">
<summary>Förklaringar av alternativen</summary>

| Alternativ | Funktion |
|---|---|
| `mkdir -p` | skapar katalogen om den saknas och ger inget fel om den redan finns |
| `wget <url>` | hämtar release-arkivet för den angivna versionen från GitHub |
| `unzip <archiv>` | packar upp arkivet |
| `-d midea_ac_lan` | målkatalog; filerna ligger i arkivet utan underkatalog och måste hamna i `custom_components/midea_ac_lan/` |

</details>

Det aktuella versionsnumret finns på projektets release-sida. Starta sedan om Home Assistant via Inställningar, System och Starta om. Varningen `We found a custom integration midea_ac_lan which has not been tested by Home Assistant` i loggen är normal för alla Custom Integrations.

## Steg 3: Lägg till PortaSplit

Gå till Inställningar, Enheter och tjänster, Lägg till integration och sök efter `Midea AC LAN`. Installationsdialogen frågar i följande ordning:

1. **Åtgärd:** `Discover automatically`.
2. **IP-adress:** `auto` söker igenom det lokala nätverket. Om PortaSplit finns i ett annat VLAN anger du dess IP-adress, eftersom sökningen via broadcast inte når över VLAN-gränser.
3. **Enhet:** PortaSplit visas som `<Geräte-ID> (Air Conditioner)`.
4. **Inloggning:** Konto, lösenord och server. För ett konto i MSmartHome-appen väljer du `SmartHome`; om inloggningen misslyckas, `NetHome Plus` med samma inloggningsuppgifter. Integrationen hämtar då token och key en gång.

Därefter visas enheten med en enda entitet `climate.<geräte-id>_climate`. Enhets-ID:t är ett 15-siffrigt tal och benämns nedan som `DEVICE_ID`.

## Steg 4: Aktivera sensorer

`Midea AC LAN` skapar som standard bara klimatentiteten. Under Inställningar, Enheter och tjänster, Midea AC LAN, Konfigurera öppnas alternativdialogen:

| Fält | Inställning |
|---|---|
| IP-adress | lämna oförändrad |
| Refresh interval | 30 sekunder (standard) |
| Sensors | välj alla tillgängliga sensorer |
| Switches | minst `Power`, `ECO Mode`, `Sleep Mode`, `Swing Vertical`, `Swing Horizontal`, `Screen Display`, `Prompt Tone` och `Fan Speed Percent` |
| Customize | lämna tomt |

Efter att du sparat skapar integrationen entiteterna utan omstart, enligt mönstret `sensor.DEVICE_ID_indoor_temperature`. En PortaSplit (enhetstyp `0xAC`, protokoll V3) gav i standby följande värden med `Midea AC LAN` v2026.9.2:

| Entitet | Betydelse | Värde i standby |
|---|---|---|
| `indoor_temperature` | rumstemperatur | 23,0 °C |
| `outdoor_temperature` | utomhustemperatur vid utomhusenheten | 23,5 °C |
| `realtime_power` | aktuell effektförbrukning | 1,5 W |
| `total_energy_consumption` | energiräknare sedan driftsättning | 90,54 kWh |
| `compressor_frequency`, `target_compressor_frequency` | kompressorfrekvens, faktisk och börvärde | 0 Hz |
| `compressor_voltage`, `compressor_current`, `compressor_power` | spänning, ström och effekt vid kompressorn | 230 V, 1 A, 4 W |
| `indoor_coil_temperature` (T2), `outdoor_coil_temperature` (T3) | förångare, kondensor | 23,5 °C |
| `discharge_pipe_temperature` (TP) | hetgasledning | 23 °C |
| `indoor_fan_speed` | fläkthastighet | 0 rpm |
| `error_code`, `full_dust` | felkod, smutsigt filter | 0, on |
| `indoor_humidity` | luftfuktighet | unknown |

PortaSplit har ingen fuktighetssensor, därför förblir `indoor_humidity` tom. Klimatentiteten har lägena Av, Auto, Kyla, Avfuktning, Värme och Fläkt, inställda temperaturer från 16 till 30 °C i steg om 0,5 °C samt fläktlägena Silent, Low, Medium, High, Full och Auto.

## Steg 5: Installera diagramkortet

Historikdiagrammen använder `apexcharts-card`. Sök efter `apexcharts-card` i HACS och hämta det. HACS registrerar kortet som en dashboard-resurs; ladda sedan om webbläsaren en gång.

## Steg 6: Konfigurera hjälpsensorer och tema

Dashboardet behöver fem hjälpsensorer: kompressor på/av, fläkthastighet som tal för stegdiagrammet, drifttid och energi sedan midnatt samt tidpunkten för senaste rapporten. Aktivera först paket och teman i `configuration.yaml`, om detta inte redan är gjort:

```yaml
homeassistant:
  packages: !include_dir_named packages

frontend:
  themes: !include_dir_merge_named themes
```

Kopiera sedan `packages/portasplit.yaml` från repositoryt till `/config/packages/` och `themes/portasplit.yaml` till `/config/themes/`, och ersätt varje `DEVICE_ID` i paketfilen med ditt eget enhets-ID. Paketet innehåller:

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
<summary>Förklaringar av alternativen</summary>

| Alternativ | Funktion |
|---|---|
| `template: binary_sensor` | kompressorn betraktas som igång när kompressorfrekvensen är över 0 Hz; om en enhet inte rapporterar frekvens gäller effekt över 150 W som kriterium |
| `template: sensor` (fläkthastighet) | översätter fläktläget till ett tal från 1 (Silent) till 6 (Auto), så att diagrammet kan rita steg |
| `states.climate['…']` | hakparentessyntax krävs eftersom objekt-ID:t börjar med en siffra; `states.climate.123…` är ogiltigt för Jinja |
| `last_reported` | tidpunkt för enhetens senaste rapport, även när inget värde har ändrats |
| `history_stats` med `type: time` | summerar tiden då kompressorsensorn var `on` |
| `start` / `end` | tidsperiod från midnatt till nu |
| `utility_meter` med `cycle: daily` | skapar en dagsräknare från totalräknaren, som återställs till 0 vid midnatt |

</details>

Kontrollera konfigurationen under Utvecklarverktyg, YAML, Kontrollera konfiguration och starta om Home Assistant; `utility_meter` kan inte läsas in via Reload. Därefter finns `binary_sensor.portasplit_kompressor`, `sensor.portasplit_lufterstufe`, `sensor.portasplit_laufzeit_heute`, `sensor.portasplit_energie_heute` och `sensor.portasplit_letzte_meldung`. Home Assistant utelämnar omljudet när entitets-ID:t skapas, alltså `lufterstufe`.

## Steg 7: Skapa dashboardet

1. Inställningar, Dashboards, Lägg till dashboard, Nytt dashboard från grunden, namn `PortaSplit`.
2. Öppna det nya dashboardet och växla till redigeringsläge via pennan.
3. Öppna Raw-konfigurationsredigeraren via menyn med tre punkter.
4. Ersätt hela innehållet med `dashboard.yaml` från repositoryt och ersätt först varje `DEVICE_ID` med ditt eget enhets-ID.
5. Spara och stäng redigeringsläget.

Dashboardet använder vyn ”Sektioner” med fyra kolumner och temat `PortaSplit Dark`. Högst upp visas åtta nyckeltal, till vänster styrningen med termostat, driftläge och fläkthastighet, och till höger historiken för de senaste 24 timmarna. Under detta följer kylkretsens tekniska värden och ett statusblock med felkod, filterstatus och senaste rapport.

Direkt efter installationen är diagrammen tomma och fylls på med tiden. `Energie heute` visar `Unbekannt`, tills energiräknaren stiger för första gången.

## Efter installationen

Token och key finns nu i Home Assistant. Om de senare inte längre kan hämtas via molnet är en säkerhetskopia enda sättet att göra en ny installation. [Del 2: Säkra PortaSplit](/blog/midea-portasplit-home-assistant-absichern) beskriver hur du säkerhetskopierar token, key och konfiguration samt isolerar PortaSplit i hemnätverket.

## Källor

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: integrationen `Midea AC LAN`: enhetsklasser som stöds, installation via HACS, lägsta version Home Assistant 2024.4.1.

2.  [midea_ac_lan: Releases](https://github.com/wuwentao/midea_ac_lan/releases): release-arkiv för installation utan HACS, testat med v2026.9.2.

3.  [midea_ac_lan: Dokumentation för klimatentiteter](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md): entiteter och attribut för luftkonditioneringsenheter, bland annat effekt, total energi och kompressorfrekvens.

4.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: integrationen `Midea Smart AC`: enhetstyperna `0xAC` och `0xCC` som stöds, PortaSplit med ”Out Silent Mode”, molnanvändning för att hämta token och key för V3-enheter samt standardport 6444.

5.  <a class="gh-badge" href="https://github.com/pfstr/ha-portasplit-dashboard" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">pfstr/ha-portasplit-dashboard</span></a>: dashboard, hjälpsensorpaket och tema från denna guide, med listan över entiteter som en PortaSplit med `Midea AC LAN` v2026.9.2 rapporterar.

6.  [apexcharts-card](https://github.com/RomRider/apexcharts-card): diagramkort för historikdiagrammen.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): installation av Custom Integrations och frontend-kort.

8.  [Home Assistant: Packages](https://www.home-assistant.io/docs/configuration/packages/): samlar Template-, Sensor- och Utility-Meter-konfiguration i en fil under `/config/packages/`.

9.  [Home Assistant: History Stats](https://www.home-assistant.io/integrations/history_stats/): sensorplattform för kompressorns drifttid sedan midnatt.

10.  [Home Assistant: Utility Meter](https://www.home-assistant.io/integrations/utility_meter/): dagsräknare baserad på totalenergimätaren.
