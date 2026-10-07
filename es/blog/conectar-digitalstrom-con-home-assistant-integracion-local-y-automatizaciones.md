---
title: "Conectar digitalSTROM con Home Assistant: integración local y automatizaciones"
navTitle: "digitalSTROM y HA"
description: "Una integración local de Home Assistant para el servidor digitalSTROM, además de luz por movimiento, modo ducha, música con 4 pulsaciones y salida automática mediante FRITZ!Box. Con los problemas surgidos en la práctica."
date: "2026-10-06"
kategorie: "Home Assistant e IoT"
timeToRead: "13 min de lectura"
themen:
  - smart-home-iot
produkte:
  - "home-assistant"
protokolle:
  - "apis"
  - "troubleshooting"
related:
  - midea-portasplit-home-assistant-einrichten
slug: "conectar-digitalstrom-con-home-assistant-integracion-local-y-automatizaciones"
translationId: "article-271967fa6d61231d"
translationOf: digitalstrom-home-assistant
url: https://rafaelpfister.ch/es/blog/conectar-digitalstrom-con-home-assistant-integracion-local-y-automatizaciones
translationSourceHash: 3eb347c3b61cdbf39b6aad2c7b4e99ca0b364c68a4c632de652726ff9bc144d0
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T10:24:52.345Z
translationReview: automatic
---

En las instalaciones de digitalSTROM, las luces, persianas y otros consumidores están conectados a bornes en el cuadro eléctrico y se controlan mediante pulsadores y un servidor digitalSTROM (dSS20). Si se añaden otros sistemas como Philips Hue, Sonos y una FRITZ!Box, resulta lógico reunirlo todo localmente en Home Assistant, sin cuenta en la nube y sin guardar la contraseña del dSS en Home Assistant. Para ello he escrito una pequeña integración y la he publicado junto con las automatizaciones correspondientes como [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local) bajo licencia MIT.

Conclusión breve: el dSS ofrece una API JSON funcional con eventos. Si se utiliza con moderación, permite llevar a Home Assistant los pulsadores, las escenas y las actividades Salir y Llegar sin retrasos, y activar así automatizaciones que digitalSTROM por sí solo no conoce.

## Cómo funciona la integración

El dSS proporciona su API por HTTPS en el puerto 8080, con un certificado autofirmado. Para los programas, digitalSTROM prevé tokens de aplicación: el programa solicita un token, el usuario lo autoriza en Configurator mediante Sistema > Autorización de acceso y, después, el programa inicia sesión con él. La contraseña del dSS permanece con el usuario.

```bash
DSS=https://dss.local:8080/json
curl -sk "$DSS/system/requestApplicationToken?applicationName=Home%20Assistant"
curl -sk "$DSS/system/loginApplication?loginToken=<APP-TOKEN>"
curl -sk "$DSS/event/subscribe?name=callScene&subscriptionID=42&token=<SESSION>"
curl -sk "$DSS/event/get?subscriptionID=42&timeout=30000&token=<SESSION>"
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-s` | sin indicador de progreso |
| `-k` | acepta el certificado autofirmado del dSS |
| `requestApplicationToken` | crea un token de aplicación que debe autorizarse en Configurator |
| `loginApplication` | intercambia el token de aplicación autorizado por un token de sesión |
| `event/subscribe` | suscribe un evento (`callScene`, `stateChange`, `buttonClick` …) con un ID elegido libremente |
| `event/get` con `timeout` | espera hasta 30 segundos nuevos eventos (Long-Poll) |

</details>

La integración mantiene permanentemente abierta una conexión Long-Poll de este tipo. Cada escena activada por un pulsador, la aplicación o Configurator llega como evento. Estados como «luz encendida en la habitación» o la presencia se encuentran en la caché del dSS bajo `/usr/states` y pueden consultarse sin cargar los bornes.

Las consultas directas de valores de salida (`device/getOutputValue`), en cambio, recorren el bus dS485 hasta el borne y tardan entre medio segundo y un segundo por valor. Muchas consultas de este tipo a intervalos cortos pueden sobrecargar los medidores. Por ello, la integración solo lee directamente las posiciones de las persianas y las salidas Joker, cada 15 minutos y aproximadamente un minuto después de un desplazamiento.

| Plataforma | Contenido |
|---|---|
| `light` | una luz por habitación, controlada mediante escenas de habitación como los pulsadores; brillo en bornes regulables, solo encendido/apagado en los conmutados |
| `cover` | persianas con subir, bajar, detener y posición |
| `scene` | ambientes nombrados en Configurator |
| `button` | Salir (escena 72) y Llegar (escena 71) |
| `binary_sensor` | presencia, alarma de viento, detectores de movimiento, salidas Joker |
| `sensor` | consumo total |

Además, la integración notifica a Home Assistant cada escena de habitación, así como Salir y Llegar, como evento `digitalstrom_local_event`. Esto permite asignar libremente 2 o 4 pulsaciones en un interruptor de luz convencional.

## Automatizaciones como Blueprints

Las siguientes automatizaciones están incluidas como Blueprints en el repositorio y pueden importarse a Home Assistant mediante el botón de importación del README.

### Luz por movimiento que no apaga la luz encendida manualmente

Cuando hay movimiento, la luz se enciende y vuelve a apagarse tras un tiempo configurable sin movimiento. Opcionalmente, esto solo se aplica por debajo de un umbral de luminosidad, dentro de una franja horaria o con atenuación nocturna. Si alguien enciende la luz con el pulsador, permanece encendida. Para ello, un interruptor auxiliar (`input_boolean`) recuerda si la automatización encendió la luz; solo entonces también la apaga.

Un detalle solo se hace evidente durante el funcionamiento: el disparador «detector inactivo durante 2 minutos» es un contador que Home Assistant descarta en cada reinicio. Si el detector registró movimiento por última vez antes del reinicio, no se produce un nuevo cambio a «inactivo» y la luz permanece encendida. Por eso, el Blueprint comprueba además cada minuto si una luz encendida por él sigue encendida aunque el detector lleve suficiente tiempo inactivo.

### Apagar la luz al salir de la habitación

El tiempo de apagado de la luz por movimiento es un compromiso: si es demasiado corto, la luz se apaga mientras alguien permanece quieto en la habitación; si es demasiado largo, sigue encendida durante minutos tras salir. Por eso, otro Blueprint utiliza un segundo detector fuera de la habitación, normalmente en el pasillo. Si este detecta movimiento, el detector de la habitación había visto movimiento poco antes (en los últimos 30 segundos) y posteriormente permanece inactivo allí, la luz de la habitación se apaga inmediatamente. Con detectores Hue, que informan de «inactividad» unos 10 segundos después del último movimiento, esto ocurre aproximadamente entre 10 y 15 segundos después de salir.

Para que nadie se quede a oscuras, la regla solo se aplica bajo ciertas condiciones: la luz debe haber sido encendida por la automatización de movimiento, el modo ducha no debe estar activo y debe haber como máximo una persona en casa. El número de personas se obtiene de los teléfonos que Home Assistant conoce como personas. La condición de 30 segundos protege además contra el caso en que alguien permanece quieto durante más tiempo en la habitación y otra persona pasa por el pasillo.

### Modo ducha con 2 pulsaciones

En la ducha, un detector de movimiento normalmente no detecta a nadie y, tras el tiempo configurado, se queda oscuro. Pulsar 2 veces el botón del baño activa en digitalSTROM el ambiente 2 (escena 17). La automatización detecta esta escena en la habitación y activa un modo ducha que bloquea el apagado. Los movimientos durante los primeros 2 minutos se ignoran (desvestirse, entrar). El primer movimiento posterior finaliza el modo ducha; a partir de entonces vuelve a aplicarse el tiempo de apagado normal. Tras una duración máxima configurable (20 minutos de forma predeterminada), termina por sí solo.

### 4 pulsaciones inician Sonos

Al pulsar varias veces rápidamente un pulsador, digitalSTROM activa los ambientes en orden: escena 5, 17, 18 y 19 en la cuarta pulsación. Si se configura la escena 19 en Configurator como «no modificar salida» para todas las lámparas, queda libre para otros fines. Un Blueprint reacciona a esta escena, apaga la luz de la habitación e inicia el altavoz Sonos de la habitación o, en habitaciones sin altavoz, varios altavoces agrupados.

El script correspondiente tiene en cuenta tres aspectos: si ya hay un altavoz en reproducción, se añade la nueva habitación a su grupo para que la reproducción permanezca sincronizada. Si no se puede reanudar nada, reproduce una emisora de radio del directorio Radio Browser. Y antes de cada inicio se establece el mismo volumen inicial; de lo contrario, una habitación suena baja y otra tan alta como la última vez que alguien escuchó música.

### Salir y llegar mediante FRITZ!Box

La integración FRITZ!Box Tools informa de si un teléfono está en la WLAN; evalúa la lista de dispositivos de la caja y cubre simultáneamente la red de 2,4 GHz, la de 5 GHz y LAN. Si ya no hay nadie en casa y no se ha pulsado Salir, un Blueprint activa Salir. Al volver a casa, sigue Llegar y, si está oscuro, una luz de bienvenida.

Como complemento, una automatización puede apagar al salir las lámparas de otros sistemas (por ejemplo, Hue) y pausar Sonos. El propio dSS apaga las lámparas digitalSTROM.

## Plano como panel de control

Una visualización clara en Home Assistant es un plano con la tarjeta integrada `picture-elements`. En cada habitación se coloca un SVG transparente sobre el plano, que se sustituye por otro ligeramente amarillo cuando la luz está encendida; al pulsar la habitación se enciende o apaga la luz. Lámparas, altavoces, persianas y detectores se muestran como símbolos en su ubicación. Las instrucciones con una configuración de ejemplo se encuentran en el repositorio bajo `docs/floor-plan-dashboard.md`.

## Problemas de la práctica

| Problema | Causa | Solución |
|---|---|---|
| El pulsador Salir no tiene efecto | El dSS no está en la red; las actividades a través de varios circuitos eléctricos se ejecutan mediante el servidor | Se restableció la conexión de red del dSS |
| Las persianas no reaccionan | La alarma de viento estaba en «activa» aunque no había sensor de viento | Se activó la escena 87 («sin viento») para todas las habitaciones |
| Las persianas no suben al salir | Escena 72 configurada como «no modificar salida» (`dontCare`) | `device/setSceneMode` con `dontCare=0`; se aceptó el valor `false`, pero se ignoró |
| No se puede regular la luz | Tubo fluorescente conectado a un borne conmutado (modo de salida 35) | La integración detecta los bornes conmutados y allí solo ofrece encendido/apagado |
| La luz permanece encendida tras reiniciar | El contador de «inactivo desde hace X minutos» se pierde al reiniciar | Comprobación adicional cada minuto |
| El teléfono se considera ausente | El iPhone utiliza una dirección WLAN privada cambiante y desconecta brevemente la WLAN en reposo | Dirección WLAN privada configurada como «Fija», período de gracia de 10 minutos |
| DECT transmite continuamente | «DECT Eco» no está disponible en cuanto se registra un dispositivo FRITZ! Smart Home | Prescindir de enchufes DECT si se desea DECT Eco |

Una búsqueda de dispositivos en el puente Hue añade todos los dispositivos que se encuentren en modo de emparejamiento en ese momento. Después, compruebe en la lista de dispositivos que solo se hayan añadido sus propias lámparas.

Las tablas de escenas de los bornes pueden leerse y modificarse mediante la API, por ejemplo con `device/getSceneMode` y `device/saveScene`. Cada una de estas llamadas pasa por el bus. Antes de realizar cambios, guarde los valores anteriores, por ejemplo en un archivo CSV, para poder restaurarlos si fuera necesario.

## Fuentes

1.  [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local): integración y Blueprints de este artículo, licencia MIT.

2.  [digitalSTROM: Manual de uso y configuración](https://www.digitalstrom.com/wp-content/uploads/2021/08/AHB_DE_A1121D001V013_neu.pdf): actividad Salir (mantener pulsado 3 segundos), ajuste «no modificar salida» en el capítulo 3.5.2.

3.  [digitalSTROM: Documentación de pulsadores de los bornes](https://www.digitalstrom.com/wp-content/uploads/2021/08/A0818D078V001_Tastendokumentation.pdf): configuraciones de fábrica de las funciones de los pulsadores según el tipo de borne.

4.  [digitalSTROM: Manuales de instrucciones](https://www.digitalstrom.com/bedienungsanleitungen/): resumen de manuales y documentación de planificación e instalación.

5.  [Home Assistant: FRITZ!Box Tools](https://www.home-assistant.io/integrations/fritz/): detección de presencia, interruptor WLAN, opciones de la integración.

6.  [Home Assistant: tarjeta Picture Elements](https://www.home-assistant.io/dashboards/picture-elements/): base del panel de plano.

7.  [Home Assistant: Blueprints](https://www.home-assistant.io/docs/automation/using_blueprints/): importación y uso de Blueprints.

8.  [Base de conocimientos de FRITZ!: el enchufe FRITZ! pierde la conexión](https://lu.fritz.com/service/wissensdatenbank/dok/FRITZ-Smart-Energy-200/3538_FRITZ-Steckdose-verliert-haufig-die-Verbindung-zur-FRITZ-Box/): indicaciones sobre el alcance y la potencia de transmisión DECT en enchufes Smart Home.

9.  [Radio Browser](https://www.radio-browser.info/): directorio libre de transmisiones de radio, integrado en Home Assistant como fuente multimedia.
