---
title: "Midea PortaSplit en Home Assistant: configuración y panel"
navTitle: "Configurar PortaSplit"
description: "Paso a paso, desde la vinculación con MSmartHome y la integración Midea AC LAN hasta el panel terminado con indicadores, control y gráficos de evolución."
date: "2026-07-24"
kategorie: "Home Assistant e IoT"
timeToRead: "10 min de lectura"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant-absichern
  - midea-v2-cloud-api-portasplit-home-assistant
slug: "controlar-midea-portasplit-localmente-con-home-assistant-y-usarla-de-forma-segura"
translationOf: "midea-portasplit-home-assistant"
translationId: article-36e7710abe426781
translationReview: automatic
translationSourceHash: 6b0bf224030d5fca539c146523bb8de015a6bab426c4e35ab116423719d9c232
translatedAt: 2026-10-09T10:53:11.544Z
translationModel: gpt-5.6-terra
image: ../images/midea-portasplit-home-assistant/portasplit-dashboard.png
url: https://rafaelpfister.ch/es/blog/controlar-midea-portasplit-localmente-con-home-assistant-y-usarla-de-forma-segura
---

La Midea PortaSplit puede controlarse directamente en la red local mediante una integración de la comunidad en Home Assistant. En siete pasos, desde la vinculación con la aplicación hasta el panel, se crea un control local con indicadores y gráficos de evolución. El panel, los sensores auxiliares y el tema están disponibles en el repositorio <a class="gh-badge" href="https://github.com/pfstr/ha-portasplit-dashboard" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">pfstr/ha-portasplit-dashboard</span></a>.

![Panel de Home Assistant de la Midea PortaSplit en modo refrigeración: indicadores arriba, termostato a 22 °C, evoluciones de la temperatura ambiente, el consumo eléctrico, la energía diaria, la frecuencia del compresor, el funcionamiento del compresor y la velocidad del ventilador; debajo, valores técnicos y estado.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

La imagen muestra el panel terminado en modo refrigeración con indicadores, control y las evoluciones de las últimas 24 horas.

La serie consta de tres partes: la parte 1 describe la configuración, la [parte 2](/blog/midea-portasplit-home-assistant-absichern) trata sobre la protección del token, la clave y la red doméstica, y la [parte 3](/blog/midea-v2-cloud-api-portasplit-home-assistant) contextualiza las advertencias sobre la API de la nube de Midea.

## Cómo funciona el control local

Tras la configuración, los comandos de control se envían directamente desde Home Assistant a la PortaSplit, sin pasar por un servidor de Midea. Sin embargo, en dispositivos con el protocolo V3, la PortaSplit solo acepta comandos locales con dos valores específicos del dispositivo: token y clave. La integración obtiene ambos una única vez durante la configuración a través de la nube de Midea y los guarda localmente:

```text
Einrichtung:   Home Assistant → Midea-Cloud → Token und Key
Betrieb:       Home Assistant → lokales Netz (6444/TCP) → PortaSplit
```

Las integraciones descritas proceden de la comunidad y no cuentan con soporte oficial ni de Midea ni de Home Assistant. Los cambios de firmware o de la nube pueden afectar a su funcionamiento.

## Qué integración elegir

Dos integraciones de la comunidad son compatibles con la PortaSplit:

| Integración | Enfoque |
|---|---|
| <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> (`Midea AC LAN`) | muchas clases de dispositivos Midea; proporciona 21 sensores para la PortaSplit, incluidos frecuencia, corriente y tensión del compresor, así como temperaturas del evaporador, condensador y gas caliente |
| <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> (`Midea Smart AC`) | orientada a equipos de aire acondicionado (`0xAC`, `0xCC`), consulta las capacidades del dispositivo y admite el modo silencioso de la unidad exterior |

Para mi PortaSplit utilizo `Midea AC LAN`; las instrucciones y el panel se basan en sus entidades. Con `Midea Smart AC` las entidades tienen otros nombres, por lo que el panel solo puede utilizarse tras adaptarlo. Ejecutar ambas integraciones simultáneamente con el mismo dispositivo provoca problemas de estado y no tiene sentido.

## Requisitos

- Midea PortaSplit con función Wi-Fi y una red Wi-Fi de 2,4 GHz
- Aplicación MSmartHome con una cuenta de Midea
- Home Assistant a partir de la versión 2024.10 (probado con 2026.7) con acceso al directorio de configuración `/config`, por ejemplo mediante el complemento File editor, Samba o SSH
- HACS para la tarjeta de gráficos `apexcharts-card`
- Acceso de red desde Home Assistant a la PortaSplit en el puerto 6444/TCP

## Paso 1: conectar la PortaSplit con MSmartHome

1. Instale la aplicación MSmartHome e inicie sesión con la cuenta de Midea.
2. Ponga la PortaSplit en modo de vinculación Wi-Fi y conéctela a la red Wi-Fi de 2,4 GHz.
3. Compruebe que puede controlar la PortaSplit mediante la aplicación.
4. Cree una reserva DHCP para la PortaSplit en el router para que conserve siempre la misma dirección IP.

Si el router utiliza el mismo SSID para 2,4 y 5 GHz, la vinculación suele funcionar de todos modos. Si hay problemas, puede ayudar crear temporalmente una red Wi-Fi independiente de 2,4 GHz.

## Paso 2: instalar Midea AC LAN

**Mediante HACS:** Abra HACS, busque `Midea AC LAN`, descargue la integración y reinicie Home Assistant.

**Sin HACS:** Descargue el archivo de la versión directamente en el directorio `custom_components`. En una instalación de Docker puede hacerlo con `docker exec -it homeassistant bash` dentro del contenedor; en Home Assistant OS, mediante el complemento Terminal:

```bash
mkdir -p /config/custom_components
cd /config/custom_components
wget https://github.com/wuwentao/midea_ac_lan/releases/download/v2026.9.2/midea_ac_lan.zip
unzip midea_ac_lan.zip -d midea_ac_lan
rm midea_ac_lan.zip
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `mkdir -p` | crea el directorio si no existe y no muestra ningún error si ya existe |
| `wget <url>` | descarga de GitHub el archivo de la versión indicada |
| `unzip <archiv>` | descomprime el archivo |
| `-d midea_ac_lan` | directorio de destino; los archivos están en el archivo sin subdirectorios y deben terminar en `custom_components/midea_ac_lan/` |

</details>

El número de versión actual figura en la página de versiones del proyecto. Después, reinicie Home Assistant mediante Ajustes, Sistema y Reiniciar. La advertencia `We found a custom integration midea_ac_lan which has not been tested by Home Assistant` en el registro es normal para cualquier integración personalizada.

## Paso 3: añadir la PortaSplit

En Ajustes, Dispositivos y servicios, Añadir integración, busque `Midea AC LAN`. El asistente de configuración pregunta, en este orden:

1. **Acción:** `Discover automatically`.
2. **Dirección IP:** `auto` busca en la red local. Si la PortaSplit está en otra VLAN, introduzca su dirección IP, ya que la búsqueda mediante difusión no atraviesa los límites entre VLAN.
3. **Dispositivo:** La PortaSplit aparece como `<Geräte-ID> (Air Conditioner)`.
4. **Inicio de sesión:** cuenta, contraseña y servidor. Para una cuenta de la aplicación MSmartHome, seleccione `SmartHome`; si el inicio de sesión falla, use `NetHome Plus` con las mismas credenciales. Con ello, la integración obtiene una única vez el token y la clave.

Después, el dispositivo aparece con una única entidad `climate.<geräte-id>_climate`. El ID del dispositivo es un número de 15 dígitos y en lo sucesivo se denomina `DEVICE_ID`.

## Paso 4: activar sensores

`Midea AC LAN` crea por defecto solo la entidad de climatización. En Ajustes, Dispositivos y servicios, Midea AC LAN, Configurar se abre el diálogo de opciones:

| Campo | Ajuste |
|---|---|
| Dirección IP | dejar sin cambios |
| Refresh interval | 30 segundos (predeterminado) |
| Sensors | seleccionar todos los sensores disponibles |
| Switches | al menos `Power`, `ECO Mode`, `Sleep Mode`, `Swing Vertical`, `Swing Horizontal`, `Screen Display`, `Prompt Tone` y `Fan Speed Percent` |
| Customize | dejar vacío |

Tras guardar, la integración crea las entidades sin reinicio, siguiendo el patrón `sensor.DEVICE_ID_indoor_temperature`. Una PortaSplit (tipo de dispositivo `0xAC`, protocolo V3) proporcionó los siguientes valores en espera con `Midea AC LAN` v2026.9.2:

| Entidad | Significado | Valor en espera |
|---|---|---|
| `indoor_temperature` | temperatura ambiente | 23,0 °C |
| `outdoor_temperature` | temperatura exterior en la unidad exterior | 23,5 °C |
| `realtime_power` | consumo eléctrico actual | 1,5 W |
| `total_energy_consumption` | contador de energía desde la puesta en servicio | 90,54 kWh |
| `compressor_frequency`, `target_compressor_frequency` | frecuencia actual y objetivo del compresor | 0 Hz |
| `compressor_voltage`, `compressor_current`, `compressor_power` | tensión, corriente y potencia en el compresor | 230 V, 1 A, 4 W |
| `indoor_coil_temperature` (T2), `outdoor_coil_temperature` (T3) | evaporador, condensador | 23,5 °C |
| `discharge_pipe_temperature` (TP) | tubería de gas caliente | 23 °C |
| `indoor_fan_speed` | velocidad del ventilador | 0 rpm |
| `error_code`, `full_dust` | código de error, filtro sucio | 0, on |
| `indoor_humidity` | humedad del aire | unknown |

La PortaSplit no dispone de sensor de humedad, por lo que `indoor_humidity` permanece vacío. La entidad de climatización incluye los modos Apagado, Auto, Refrigeración, Deshumidificación, Calefacción y Ventilación, temperaturas objetivo de 16 a 30 °C en incrementos de 0,5 °C, y las velocidades de ventilador Silent, Low, Medium, High, Full y Auto.

## Paso 5: instalar la tarjeta de gráficos

Los gráficos de evolución utilizan `apexcharts-card`. Busque `apexcharts-card` en HACS y descárguela. HACS registra la tarjeta como recurso del panel; después, recargue el navegador una vez.

## Paso 6: configurar los sensores auxiliares y el tema

El panel necesita cinco sensores auxiliares: compresor encendido/apagado, velocidad del ventilador como número para el gráfico escalonado, tiempo de funcionamiento y energía desde medianoche, y la hora del último informe. Primero active los paquetes y temas en `configuration.yaml`, si todavía no lo ha hecho:

```yaml
homeassistant:
  packages: !include_dir_named packages

frontend:
  themes: !include_dir_merge_named themes
```

A continuación, copie `packages/portasplit.yaml` del repositorio a `/config/packages/` y `themes/portasplit.yaml` a `/config/themes/`, y sustituya en el archivo del paquete cada `DEVICE_ID` por el ID de su dispositivo. El paquete contiene:

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
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `template: binary_sensor` | el compresor se considera en funcionamiento cuando su frecuencia supera 0 Hz; si un dispositivo no informa de la frecuencia, una potencia superior a 150 W se considera el criterio |
| `template: sensor` (velocidad del ventilador) | traduce el modo de ventilador a un número de 1 (Silent) a 6 (Auto), para que el gráfico pueda dibujar escalones |
| `states.climate['…']` | la notación entre corchetes es necesaria porque el ID de objeto empieza por un número; `states.climate.123…` no es válido para Jinja |
| `last_reported` | hora del último informe del dispositivo, aunque no haya cambiado ningún valor |
| `history_stats` con `type: time` | suma el tiempo durante el que el sensor del compresor estuvo `on` |
| `start` / `end` | periodo desde medianoche hasta ahora |
| `utility_meter` con `cycle: daily` | crea un contador diario a partir del contador total, que se restablece a 0 a medianoche |

</details>

Compruebe la configuración en Herramientas para desarrolladores, YAML, Comprobar configuración y reinicie Home Assistant; `utility_meter` no se puede cargar mediante recarga. Después existirán `binary_sensor.portasplit_kompressor`, `sensor.portasplit_lufterstufe`, `sensor.portasplit_laufzeit_heute`, `sensor.portasplit_energie_heute` y `sensor.portasplit_letzte_meldung`. Home Assistant omite la diéresis al crear el ID de entidad, de ahí `lufterstufe`.

## Paso 7: crear el panel

1. Ajustes, Paneles, Añadir panel, Nuevo panel desde cero, nombre `PortaSplit`.
2. Abra el nuevo panel y pase al modo de edición mediante el icono del lápiz.
3. Abra el editor de configuración sin procesar desde el menú de tres puntos.
4. Sustituya por completo el contenido por `dashboard.yaml` del repositorio; antes, sustituya cada `DEVICE_ID` por el ID de su dispositivo.
5. Guarde y cierre el modo de edición.

El panel utiliza la vista «Secciones» con cuatro columnas y el tema `PortaSplit Dark`. En la parte superior hay ocho indicadores; a la izquierda, los controles con termostato, modo de funcionamiento y velocidad del ventilador; a la derecha, las evoluciones de las últimas 24 horas. Debajo se encuentran los valores técnicos del circuito frigorífico y un bloque de estado con código de error, estado del filtro y último informe.

Justo después de la configuración, los gráficos están vacíos y se llenan con el tiempo de funcionamiento. `Energie heute` muestra `Unbekannt` hasta que el contador de energía aumente por primera vez.

## Después de la configuración

El token y la clave están ahora en Home Assistant. Si más adelante ya no pueden obtenerse desde la nube, una copia de seguridad será la única forma de realizar una nueva configuración. La [parte 2: proteger la PortaSplit](/blog/midea-portasplit-home-assistant-absichern) explica cómo proteger el token, la clave y la configuración, y cómo aislar la PortaSplit en la red doméstica.

## Fuentes

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: integración `Midea AC LAN`: clases de dispositivos compatibles, instalación mediante HACS, versión mínima de Home Assistant 2024.4.1.

2.  [midea_ac_lan: Releases](https://github.com/wuwentao/midea_ac_lan/releases): archivos de versiones para la instalación sin HACS, probado con v2026.9.2.

3.  [midea_ac_lan: documentación de las entidades de climatización](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md): entidades y atributos para equipos de aire acondicionado, incluidos potencia, energía total y frecuencia del compresor.

4.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: integración `Midea Smart AC`: tipos de dispositivos compatibles `0xAC` y `0xCC`, PortaSplit con «Out Silent Mode», uso de la nube para obtener el token y la clave en dispositivos V3, y puerto predeterminado 6444.

5.  <a class="gh-badge" href="https://github.com/pfstr/ha-portasplit-dashboard" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">pfstr/ha-portasplit-dashboard</span></a>: panel, paquete de sensores auxiliares y tema de estas instrucciones, con la lista de entidades que informa una PortaSplit con `Midea AC LAN` v2026.9.2.

6.  [apexcharts-card](https://github.com/RomRider/apexcharts-card): tarjeta de gráficos para los gráficos de evolución.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): instalación de integraciones personalizadas y tarjetas de frontend.

8.  [Home Assistant: Packages](https://www.home-assistant.io/docs/configuration/packages/): agrupación de configuración de plantillas, sensores y Utility Meter en un archivo bajo `/config/packages/`.

9.  [Home Assistant: History Stats](https://www.home-assistant.io/integrations/history_stats/): plataforma de sensores para el tiempo de funcionamiento del compresor desde medianoche.

10.  [Home Assistant: Utility Meter](https://www.home-assistant.io/integrations/utility_meter/): contador diario basado en el contador total de energía.
