---
title: "Conectar digitalSTROM con Home Assistant: integración local y automatizaciones"
navTitle: "digitalSTROM y HA"
description: "Una integración local de Home Assistant para el servidor digitalSTROM, además de iluminación por movimiento, modo ducha, música con 4× pulsaciones y salida automática mediante FRITZ!Box. Con los problemas surgidos en la práctica."
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
  - midea-portasplit-home-assistant
slug: "conectar-digitalstrom-con-home-assistant-integracion-local-y-automatizaciones"
translationId: "article-271967fa6d61231d"
translationOf: digitalstrom-home-assistant
translationSourceHash: 162e47131f3dd2ebe7da91e342e930e442e8b0130eb234edbce882409781e07b
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T09:23:33.798Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/conectar-digitalstrom-con-home-assistant-integracion-local-y-automatizaciones
---

En las instalaciones de digitalSTROM, las luces, persianas y otros consumidores están conectados a bornes en el cuadro eléctrico y se controlan mediante pulsadores y un servidor digitalSTROM (dSS20). Cuando se añaden otros sistemas como Philips Hue, Sonos y una FRITZ!Box, resulta natural centralizarlo todo localmente en Home Assistant, sin cuenta en la nube y sin guardar la contraseña del dSS en Home Assistant. Para ello escribí una pequeña integración y la publiqué, junto con las automatizaciones correspondientes, como [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local) bajo licencia MIT.

Conclusión breve: el dSS ofrece una API JSON funcional con eventos. Quien la utilice con moderación obtiene pulsadores, escenas y las actividades Salir y Llegar en Home Assistant sin demora, y puede activar con ellas automatizaciones que digitalSTROM por sí solo no conoce.

## Cómo funciona la integración

El dSS ofrece su API mediante HTTPS en el puerto 8080, con un certificado autofirmado. digitalSTROM prevé tokens de aplicación para los programas: el programa solicita un token, el usuario lo autoriza en el Configurator en Sistema > Permiso de acceso y, después, el programa inicia sesión con él. La contraseña del dSS permanece con el usuario.

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
| `-k` | aceptar el certificado autofirmado del dSS |
| `requestApplicationToken` | crea un token de aplicación que debe autorizarse en el Configurator |
| `loginApplication` | intercambia el token de aplicación autorizado por un token de sesión |
| `event/subscribe` | suscribe un evento (`callScene`, `stateChange`, `buttonClick` …) bajo un ID elegido libremente |
| `event/get` con `timeout` | espera hasta 30 segundos nuevos eventos (sondeo largo) |

</details>

La integración mantiene abierta permanentemente una conexión de sondeo largo de este tipo. Cada escena activada por un pulsador, la aplicación o el Configurator llega como evento. Estados como «luz encendida en la habitación» o la presencia se encuentran en la caché del dSS, en `/usr/states`, y pueden consultarse sin cargar los bornes.

En cambio, las consultas directas de valores de salida (`device/getOutputValue`) pasan por el bus dS485 hasta el borne y tardan entre medio segundo y un segundo por valor. Muchas consultas de este tipo a intervalos cortos pueden sobrecargar los Meter. Por ello, la integración solo lee directamente las posiciones de persianas y las salidas Joker, cada 15 minutos y aproximadamente un minuto después de un movimiento.

| Plataforma | Contenido |
|---|---|
| `light` | una luz por habitación, controlada mediante escenas de habitación como los pulsadores; brillo en bornes regulables, solo encendido/apagado en los conmutados |
| `cover` | persianas con subir, bajar, detener y posición |
| `scene` | ambientes nombrados en el Configurator |
| `button` | Salir (escena 72) y Llegar (escena 71) |
| `binary_sensor` | presencia, alarma de viento, detectores de movimiento, salidas Joker |
| `sensor` | consumo total |

Además, la integración comunica cada escena de habitación, así como Salir y Llegar, como evento `digitalstrom_local_event` a Home Assistant. Así se pueden asignar libremente 2× o 4× pulsaciones en un interruptor de luz convencional.

## Automatizaciones como Blueprints

Las siguientes automatizaciones están incluidas como Blueprints en el repositorio y pueden incorporarse a Home Assistant mediante el botón de importación del README.

### Iluminación por movimiento que no apaga la luz encendida manualmente

Con movimiento, la luz se enciende y, tras un tiempo configurable sin movimiento, vuelve a apagarse. Opcionalmente, esto solo se aplica por debajo de un umbral de luminosidad, dentro de una franja horaria o con atenuación nocturna. Si alguien enciende la luz con el pulsador, permanece encendida. Un interruptor auxiliar (`input_boolean`) registra si la automatización encendió la luz; solo entonces también la apaga.

Un detalle solo se aprecia durante el funcionamiento: el activador «detector en reposo durante 2 minutos» es un contador que Home Assistant descarta en cada reinicio. Si el detector se movió por última vez antes del reinicio, no se produce un nuevo cambio a «en reposo» y la luz permanece encendida. Por eso, el Blueprint comprueba adicionalmente cada minuto si una luz encendida por él sigue iluminada aunque el detector lleve suficiente tiempo en reposo.

### Apagar la luz al salir de la habitación

El tiempo de apagado de la iluminación por movimiento es un compromiso: si es demasiado corto, la luz se apaga mientras alguien permanece quieto en la habitación; si es demasiado largo, sigue encendida durante minutos tras salir. Por ello, otro Blueprint utiliza un segundo detector fuera de la habitación, normalmente en el pasillo. Si este detecta movimiento, el detector de la habitación había detectado movimiento poco antes (dentro de 30 segundos) y después permanece en reposo, la luz de la habitación se apaga inmediatamente. Con detectores Hue, que notifican «en reposo» unos 10 segundos después del último movimiento, ocurre aproximadamente entre 10 y 15 segundos después de salir.

Para que nadie se quede a oscuras, la regla solo se aplica bajo ciertas condiciones: la luz debe haber sido encendida por la automatización de movimiento, no puede estar activo un modo ducha y debe haber como máximo una persona en casa. El número de personas se determina a partir de los teléfonos que Home Assistant reconoce como personas. La condición de 30 segundos protege además frente al caso de que alguien permanezca quieto durante más tiempo en la habitación y otra persona pase por el pasillo.

### Modo ducha mediante 2× pulsaciones

En la ducha, un detector de movimiento normalmente no detecta a nadie y, tras el tiempo configurado, se hace de noche. 2× pulsaciones en el pulsador del baño activan en digitalSTROM el ambiente 2 (escena 17). La automatización reconoce esta escena en la habitación y activa un modo ducha que bloquea el apagado. Los movimientos durante los primeros 2 minutos se ignoran (desvestirse, entrar). El primer movimiento posterior finaliza el modo ducha; a partir de entonces vuelve a aplicarse el tiempo de apagado normal. Tras una duración máxima configurable (20 minutos por defecto), finaliza por sí solo.

### 4× pulsaciones inician Sonos

Al pulsar varias veces rápidamente un pulsador, digitalSTROM cambia los ambientes uno tras otro: escena 5, 17, 18 y, en la cuarta pulsación, 19. Si se configura la escena 19 en el Configurator para todas las lámparas como «No modificar salida», queda libre para otros fines. Un Blueprint reacciona a esta escena, apaga la luz de la habitación e inicia el altavoz Sonos de la habitación o, en habitaciones sin altavoz, varios conjuntamente.

El script correspondiente tiene en cuenta tres aspectos: si ya hay un altavoz reproduciendo, se añade la nueva habitación a su grupo para mantener la reproducción sincronizada. Si no se puede reanudar nada, reproduce una emisora de radio del directorio Radio Browser. Además, antes de cada inicio se establece el mismo volumen inicial; de lo contrario, una habitación reproduce bajo y otra al volumen al que alguien escuchó música por última vez.

### Salir y Llegar mediante FRITZ!Box

La integración FRITZ!Box Tools informa de si un teléfono está en la WLAN; evalúa la lista de dispositivos del router y cubre simultáneamente las redes de 2,4 GHz, 5 GHz y LAN. Si ya no hay nadie en casa y no se ha pulsado Salir, un Blueprint activa Salir. Al volver a casa, sigue Llegar y, si está oscuro, una luz de bienvenida.

Como complemento, una automatización puede apagar las lámparas de otros sistemas (por ejemplo, Hue) y pausar Sonos al salir. El propio dSS apaga las lámparas digitalSTROM.

## Plano como panel de control

Una representación clara en Home Assistant es un plano con la tarjeta integrada `picture-elements`. Sobre el plano se coloca un SVG transparente por habitación, que se sustituye por otro ligeramente amarillo cuando la luz está encendida; al pulsar la habitación se enciende o apaga la luz. Lámparas, altavoces, persianas y detectores se sitúan como símbolos en su ubicación. Las instrucciones con una configuración de ejemplo se encuentran en el repositorio, en `docs/floor-plan-dashboard.md`.

## Problemas de la práctica

| Problema | Causa | Solución |
|---|---|---|
| El pulsador Salir no tiene efecto | El dSS no está en la red; las actividades a través de varios circuitos eléctricos pasan por el servidor | Se restauró la conexión de red del dSS |
| Las persianas no reaccionan | La alarma de viento estaba «activa» aunque no había sensor de viento | Se activó la escena 87 («sin viento») para todas las habitaciones |
| Las persianas no suben al salir | Escena 72 configurada como «No modificar salida» (`dontCare`) | `device/setSceneMode` con `dontCare=0`; se aceptó el valor `false`, pero se ignoró |
| No se puede regular la luz | Tubo fluorescente conectado a un borne conmutado (modo de salida 35) | La integración reconoce los bornes conmutados y solo ofrece encendido/apagado en ellos |
| La luz permanece encendida tras reiniciar | El contador de «en reposo desde hace X minutos» se pierde al reiniciar | Comprobación adicional cada minuto |
| El teléfono se considera ausente | El iPhone utiliza una dirección WLAN privada cambiante y desconecta brevemente la WLAN en reposo | Dirección WLAN privada en «Fija», periodo de gracia de 10 minutos |
| DECT transmite continuamente | «DECT Eco» no está disponible en cuanto se registra un dispositivo FRITZ! Smart Home | Prescindir de enchufes DECT si se desea DECT Eco |

Una búsqueda de dispositivos en el puente Hue incorpora todos los dispositivos que se encuentren en ese momento en modo de emparejamiento. Después, compruebe en la lista de dispositivos que solo se hayan añadido sus propias lámparas.

Las tablas de escenas de los bornes pueden leerse y modificarse mediante la API, por ejemplo con `device/getSceneMode` y `device/saveScene`. Cada una de estas llamadas pasa por el bus. Antes de realizar cambios, guarde los valores anteriores, por ejemplo en un archivo CSV, para poder restaurarlos si fuera necesario.

## Historial de cambios

- **25 de octubre de 2026:** se actualizó el enlace de Fuentes a la nueva dirección de la base de conocimientos de FRITZ!.

## Fuentes

1.  [pfstr/ha-digitalstrom-local](https://github.com/pfstr/ha-digitalstrom-local): integración y Blueprints de este artículo, licencia MIT.

2.  [digitalSTROM: Manual de uso y configuración](https://www.digitalstrom.com/wp-content/uploads/2021/08/AHB_DE_A1121D001V013_neu.pdf): actividad Salir (mantener pulsado 3 segundos), ajuste «No modificar salida» en el capítulo 3.5.2.

3.  [digitalSTROM: Documentación de pulsadores de los bornes](https://www.digitalstrom.com/wp-content/uploads/2021/08/A0818D078V001_Tastendokumentation.pdf): ajustes de fábrica de las funciones de pulsador por tipo de borne.

4.  [digitalSTROM: Manuales de instrucciones](https://www.digitalstrom.com/bedienungsanleitungen/): resumen de manuales, documentación para planificación e instalación.

5.  [Home Assistant: FRITZ!Box Tools](https://www.home-assistant.io/integrations/fritz/): detección de presencia, interruptores WLAN, opciones de la integración.

6.  [Home Assistant: tarjeta Picture Elements](https://www.home-assistant.io/dashboards/picture-elements/): base del panel de control con plano.

7.  [Home Assistant: Blueprints](https://www.home-assistant.io/docs/automation/using_blueprints/): importación y uso de Blueprints.

8.  [Base de conocimientos de FRITZ!: el enchufe FRITZ! pierde la conexión](https://fritz.com/de-lu/apps/knowledge-base/FRITZ-Smart-Energy-200/3538_FRITZ-Steckdose-verliert-haufig-die-Verbindung-zur-FRITZ-Box/): indicaciones sobre el alcance y la potencia de radio DECT en enchufes Smart Home.

9.  [Radio Browser](https://www.radio-browser.info/): directorio libre de transmisiones de radio, integrado en Home Assistant como fuente multimedia.
