---
title: "Midea PortaSplit in Home Assistant: configurazione e dashboard"
navTitle: "Configurare PortaSplit"
description: "Passo dopo passo, dall'associazione con MSmartHome all'integrazione Midea AC LAN fino alla dashboard pronta con indicatori, controlli e grafici storici."
date: "2026-07-24"
kategorie: "Home Assistant e IoT"
timeToRead: "10 min di lettura"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant-absichern
  - midea-v2-cloud-api-portasplit-home-assistant
slug: "controllare-localmente-e-utilizzare-in-sicurezza-midea-portasplit-con-home-assistant"
translationOf: "midea-portasplit-home-assistant"
translationId: article-36e7710abe426781
translationReview: required
translationSourceHash: 6b0bf224030d5fca539c146523bb8de015a6bab426c4e35ab116423719d9c232
translatedAt: 2026-10-09T10:52:09.554Z
translationModel: gpt-5.6-terra
image: ../images/midea-portasplit-home-assistant/portasplit-dashboard.png
url: https://rafaelpfister.ch/it/blog/controllare-localmente-e-utilizzare-in-sicurezza-midea-portasplit-con-home-assistant
---

Midea PortaSplit può essere controllato direttamente nella rete locale tramite Home Assistant con un'integrazione della community. In sette passaggi si realizza un controllo locale, dall'associazione con l'app fino alla dashboard con indicatori e grafici storici. La dashboard, i sensori di supporto e il tema sono disponibili nel repository <a class="gh-badge" href="https://github.com/pfstr/ha-portasplit-dashboard" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">pfstr/ha-portasplit-dashboard</span></a>.

![Dashboard di Home Assistant della Midea PortaSplit in modalità raffreddamento: indicatori in alto, termostato a 22 °C, andamenti di temperatura ambiente, assorbimento di potenza, energia giornaliera, frequenza del compressore, funzionamento del compressore e velocità della ventola, sotto valori tecnici e stato.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

L'illustrazione mostra la dashboard completata in modalità raffreddamento con indicatori, controlli e andamenti delle ultime 24 ore.

La serie si compone di tre parti: la parte 1 descrive la configurazione, [la parte 2](/blog/midea-portasplit-home-assistant-absichern) tratta la protezione di token, key e rete domestica, [la parte 3](/blog/midea-v2-cloud-api-portasplit-home-assistant) contestualizza gli avvisi relativi all'API cloud di Midea.

## Come funziona il controllo locale

Dopo la configurazione, i comandi di controllo passano direttamente da Home Assistant alla PortaSplit, senza transitare da un server Midea. Per i dispositivi con protocollo V3, tuttavia, PortaSplit accetta comandi locali solo con due valori specifici del dispositivo: token e key. Durante la configurazione, l'integrazione recupera entrambi una sola volta dal cloud Midea e li salva localmente:

```text
Einrichtung:   Home Assistant → Midea-Cloud → Token und Key
Betrieb:       Home Assistant → lokales Netz (6444/TCP) → PortaSplit
```

Le integrazioni descritte provengono dalla community e non sono supportate ufficialmente né da Midea né da Home Assistant. Modifiche al firmware o al cloud possono influenzarne il comportamento.

## Quale integrazione scegliere

Due integrazioni della community supportano PortaSplit:

| Integrazione | Focus |
|---|---|
| <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> (`Midea AC LAN`) | molte classi di dispositivi Midea; per PortaSplit fornisce 21 sensori, tra cui frequenza, corrente e tensione del compressore, nonché temperature dell'evaporatore, del condensatore e del gas caldo |
| <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> (`Midea Smart AC`) | pensata per climatizzatori (`0xAC`, `0xCC`), interroga le capacità del dispositivo e supporta la modalità silenziosa dell'unità esterna |

Per la mia PortaSplit utilizzo `Midea AC LAN`; guida e dashboard si basano sulle sue entità. Con `Midea Smart AC` le entità hanno nomi diversi, quindi la dashboard può essere utilizzata solo dopo gli opportuni adattamenti. Usare entrambe le integrazioni contemporaneamente con lo stesso dispositivo causa problemi di stato e non ha senso.

## Prerequisiti

- Midea PortaSplit con funzione Wi-Fi e una rete Wi-Fi a 2,4 GHz
- App MSmartHome con account Midea
- Home Assistant dalla versione 2024.10 (testato con 2026.7) con accesso alla directory di configurazione `/config`, ad esempio tramite l'add-on File editor, Samba o SSH
- HACS per la scheda dei grafici `apexcharts-card`
- Accesso di rete da Home Assistant a PortaSplit sulla porta 6444/TCP

## Passaggio 1: collegare PortaSplit a MSmartHome

1. Installare l'app MSmartHome e accedere con l'account Midea.
2. Mettere PortaSplit in modalità di associazione Wi-Fi e collegarla alla rete Wi-Fi a 2,4 GHz.
3. Verificare che PortaSplit possa essere controllata tramite l'app.
4. Creare una prenotazione DHCP nel router per PortaSplit, affinché riceva sempre lo stesso indirizzo IP.

Se il router usa lo stesso SSID per 2,4 e 5 GHz, l'associazione di solito funziona comunque. In caso di problemi, può essere utile usare temporaneamente una rete Wi-Fi separata a 2,4 GHz.

## Passaggio 2: installare Midea AC LAN

**Tramite HACS:** aprire HACS, cercare `Midea AC LAN`, scaricare l'integrazione e riavviare Home Assistant.

**Senza HACS:** scaricare l'archivio della release direttamente nella directory `custom_components`. In un'installazione Docker, ciò è possibile con `docker exec -it homeassistant bash` nel container; in Home Assistant OS, tramite l'add-on Terminal:

```bash
mkdir -p /config/custom_components
cd /config/custom_components
wget https://github.com/wuwentao/midea_ac_lan/releases/download/v2026.9.2/midea_ac_lan.zip
unzip midea_ac_lan.zip -d midea_ac_lan
rm midea_ac_lan.zip
```

<details class="options-details">
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `mkdir -p` | crea la directory se manca e non segnala errori se esiste già |
| `wget <url>` | scarica da GitHub l'archivio della release della versione indicata |
| `unzip <archiv>` | estrae l'archivio |
| `-d midea_ac_lan` | directory di destinazione; nell'archivio i file sono senza sottodirectory e devono finire in `custom_components/midea_ac_lan/` |

</details>

Il numero di versione attuale è indicato nella pagina delle release del progetto. Riavviare quindi Home Assistant tramite Impostazioni, Sistema e Riavvia. L'avviso `We found a custom integration midea_ac_lan which has not been tested by Home Assistant` nel registro è normale per ogni Custom Integration.

## Passaggio 3: aggiungere PortaSplit

In Impostazioni, Dispositivi e servizi, Aggiungi integrazione, cercare `Midea AC LAN`. La procedura guidata di configurazione richiede in sequenza:

1. **Azione:** `Discover automatically`.
2. **Indirizzo IP:** `auto` cerca nella rete locale. Se PortaSplit si trova in una VLAN diversa, inserire il suo indirizzo IP, poiché la ricerca tramite broadcast non supera i confini delle VLAN.
3. **Dispositivo:** PortaSplit appare come `<Geräte-ID> (Air Conditioner)`.
4. **Accesso:** account, password e server. Per un account dell'app MSmartHome, selezionare `SmartHome`; se l'accesso non riesce, provare `NetHome Plus` con le stesse credenziali. L'integrazione recupera così token e key una sola volta.

Successivamente il dispositivo appare con una sola entità `climate.<geräte-id>_climate`. L'ID dispositivo è un numero di 15 cifre e nel seguito viene indicato come `DEVICE_ID`.

## Passaggio 4: abilitare i sensori

`Midea AC LAN` crea per impostazione predefinita solo l'entità del climatizzatore. In Impostazioni, Dispositivi e servizi, Midea AC LAN, Configura si apre la finestra delle opzioni:

| Campo | Impostazione |
|---|---|
| Indirizzo IP | lasciare invariato |
| Refresh interval | 30 secondi (predefinito) |
| Sensors | selezionare tutti i sensori disponibili |
| Switches | almeno `Power`, `ECO Mode`, `Sleep Mode`, `Swing Vertical`, `Swing Horizontal`, `Screen Display`, `Prompt Tone` e `Fan Speed Percent` |
| Customize | lasciare vuoto |

Dopo il salvataggio, l'integrazione crea le entità senza riavvio, secondo lo schema `sensor.DEVICE_ID_indoor_temperature`. Una PortaSplit (tipo di dispositivo `0xAC`, protocollo V3) con `Midea AC LAN` v2026.9.2 in standby ha fornito questi valori:

| Entità | Significato | Valore in standby |
|---|---|---|
| `indoor_temperature` | temperatura ambiente | 23,0 °C |
| `outdoor_temperature` | temperatura esterna presso l'unità esterna | 23,5 °C |
| `realtime_power` | assorbimento di potenza attuale | 1,5 W |
| `total_energy_consumption` | contatore energetico dalla messa in servizio | 90,54 kWh |
| `compressor_frequency`, `target_compressor_frequency` | frequenza effettiva e richiesta del compressore | 0 Hz |
| `compressor_voltage`, `compressor_current`, `compressor_power` | tensione, corrente e potenza del compressore | 230 V, 1 A, 4 W |
| `indoor_coil_temperature` (T2), `outdoor_coil_temperature` (T3) | evaporatore, condensatore | 23,5 °C |
| `discharge_pipe_temperature` (TP) | linea del gas caldo | 23 °C |
| `indoor_fan_speed` | velocità della ventola | 0 rpm |
| `error_code`, `full_dust` | codice di errore, filtro sporco | 0, on |
| `indoor_humidity` | umidità dell'aria | unknown |

PortaSplit non dispone di un sensore di umidità, pertanto `indoor_humidity` rimane vuoto. L'entità del climatizzatore conosce le modalità Spento, Auto, Raffreddamento, Deumidificazione, Riscaldamento e Ventilazione, temperature impostate da 16 a 30 °C in incrementi di 0,5 °C e le velocità della ventola Silent, Low, Medium, High, Full e Auto.

## Passaggio 5: installare la scheda dei grafici

I grafici storici utilizzano `apexcharts-card`. In HACS cercare `apexcharts-card` e scaricarla. HACS registra la scheda come risorsa della dashboard; quindi ricaricare una volta il browser.

## Passaggio 6: configurare i sensori di supporto e il tema

La dashboard richiede cinque sensori di supporto: compressore acceso/spento, velocità della ventola come numero per il grafico a gradini, tempo di funzionamento ed energia dalla mezzanotte, nonché l'orario dell'ultimo messaggio. Per prima cosa abilitare pacchetti e temi in `configuration.yaml`, se non è già stato fatto:

```yaml
homeassistant:
  packages: !include_dir_named packages

frontend:
  themes: !include_dir_merge_named themes
```

Quindi copiare dal repository `packages/portasplit.yaml` in `/config/packages/` e `themes/portasplit.yaml` in `/config/themes/`, e nella file del pacchetto sostituire ogni `DEVICE_ID` con il proprio ID dispositivo. Il pacchetto contiene:

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
<summary>Opzioni spiegate</summary>

| Opzione | Effetto |
|---|---|
| `template: binary_sensor` | il compressore è considerato in funzione quando la sua frequenza supera 0 Hz; se un dispositivo non segnala la frequenza, il criterio è una potenza superiore a 150 W |
| `template: sensor` (velocità della ventola) | traduce la modalità della ventola in un numero da 1 (Silent) a 6 (Auto), affinché il grafico possa disegnare i gradini |
| `states.climate['…']` | la notazione tra parentesi è necessaria perché l'ID oggetto inizia con cifre; `states.climate.123…` non è valido per Jinja |
| `last_reported` | orario dell'ultimo messaggio del dispositivo, anche quando nessun valore è cambiato |
| `history_stats` con `type: time` | somma il tempo in cui il sensore del compressore era `on` |
| `start` / `end` | intervallo dalla mezzanotte fino a ora |
| `utility_meter` con `cycle: daily` | crea dal contatore totale un contatore giornaliero che viene azzerato a mezzanotte |

</details>

Verificare la configurazione in Strumenti per sviluppatori, YAML, Verifica configurazione e riavviare Home Assistant; `utility_meter` non può essere caricato tramite Reload. In seguito esistono `binary_sensor.portasplit_kompressor`, `sensor.portasplit_lufterstufe`, `sensor.portasplit_laufzeit_heute`, `sensor.portasplit_energie_heute` e `sensor.portasplit_letzte_meldung`. Home Assistant omette l'umlaut durante la creazione dell'ID dell'entità, quindi `lufterstufe`.

## Passaggio 7: creare la dashboard

1. Impostazioni, Dashboard, Aggiungi dashboard, Nuova dashboard da zero, nome `PortaSplit`.
2. Aprire la nuova dashboard e passare alla modalità di modifica tramite l'icona della matita.
3. Aprire l'editor di configurazione raw dal menu con i tre punti.
4. Sostituire interamente il contenuto con `dashboard.yaml` dal repository, sostituendo prima ogni `DEVICE_ID` con il proprio ID dispositivo.
5. Salvare e uscire dalla modalità di modifica.

La dashboard utilizza la vista «Sezioni» con quattro colonne e il tema `PortaSplit Dark`. In alto sono presenti otto indicatori, a sinistra i controlli con termostato, modalità di funzionamento e velocità della ventola, a destra gli andamenti delle ultime 24 ore. Sotto seguono i valori tecnici del circuito frigorifero e un blocco di stato con codice di errore, stato del filtro e ultimo messaggio.

Subito dopo la configurazione i grafici sono vuoti e si riempiono nel tempo. `Energie heute` mostra `Unbekannt`, finché il contatore energetico non aumenta per la prima volta.

## Dopo la configurazione

Token e key si trovano ora in Home Assistant. Se in futuro non fosse più possibile recuperarli dal cloud, un backup è l'unica via per una nuova configurazione. La [parte 2: proteggere PortaSplit](/blog/midea-portasplit-home-assistant-absichern) descrive come eseguire il backup di token, key e configurazione e isolare PortaSplit nella rete domestica.

## Fonti

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: integrazione `Midea AC LAN`: classi di dispositivi supportate, installazione tramite HACS, versione minima di Home Assistant 2024.4.1.

2.  [midea_ac_lan: Release](https://github.com/wuwentao/midea_ac_lan/releases): archivi delle release per l'installazione senza HACS, testato con v2026.9.2.

3.  [midea_ac_lan: documentazione delle entità del climatizzatore](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md): entità e attributi per climatizzatori, tra cui potenza, energia totale e frequenza del compressore.

4.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: integrazione `Midea Smart AC`: tipi di dispositivo supportati `0xAC` e `0xCC`, PortaSplit con «Out Silent Mode», uso del cloud per ottenere token e key sui dispositivi V3 e porta standard 6444.

5.  <a class="gh-badge" href="https://github.com/pfstr/ha-portasplit-dashboard" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">pfstr/ha-portasplit-dashboard</span></a>: dashboard, pacchetto di sensori di supporto e tema di questa guida, con l'elenco delle entità segnalate da una PortaSplit con `Midea AC LAN` v2026.9.2.

6.  [apexcharts-card](https://github.com/RomRider/apexcharts-card): scheda dei grafici per i grafici storici.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): installazione di Custom Integrations e schede frontend.

8.  [Home Assistant: Packages](https://www.home-assistant.io/docs/configuration/packages/): raggruppamento della configurazione di template, sensori e Utility Meter in un file sotto `/config/packages/`.

9.  [Home Assistant: History Stats](https://www.home-assistant.io/integrations/history_stats/): piattaforma di sensori per il tempo di funzionamento del compressore dalla mezzanotte.

10.  [Home Assistant: Utility Meter](https://www.home-assistant.io/integrations/utility_meter/): contatore giornaliero basato sul contatore energetico totale.
