---
title: "Home Assistant 2026.11.0: Marketplace en lugar de HACS y nuevos nombres de entidades"
navTitle: "Home Assistant 2026.11.0"
description: "Home Assistant 2026.11.0 integra HACS como Marketplace, recompone los nombres de las entidades a partir del dispositivo y la entidad, e incorpora numerosos cambios incompatibles con versiones anteriores. Antes de actualizar, revise los paneles, las plantillas y la detección de presencia mediante la FRITZ!Box."
date: "2026-11-05"
kategorie: "Home Assistant e IoT"
timeToRead: "6 min de lectura"
themen:
  - smart-home-iot
produkte:
  - "home-assistant"
protokolle:
  - "releases"
slug: "home-assistant-2026-11-0-marketplace-en-lugar-de-hacs-y-nuevos-nombres-de-entidades"
translationId: article-6fe587829541b0ff
translationOf: home-assistant-2026-11-0
url: https://rafaelpfister.ch/es/blog/home-assistant-2026-11-0-marketplace-en-lugar-de-hacs-y-nuevos-nombres-de-entidades
translationSourceHash: e560e2600ff8f900817c5c323e9f6a0c0bb7be09065e7794e695e6fbe12d1f52
translationModel: gpt-5.6-terra
translatedAt: 2026-11-05T14:24:52.806Z
translationReview: automatic
---

# Home Assistant 2026.11.0: Marketplace en lugar de HACS y nuevos nombres de entidades

Home Assistant 2026.11.0 se publicó el 4 de noviembre de 2026 y sucede a la versión 2026.10.4. Esta versión integra el anterior Home Assistant Community Store (HACS) como Marketplace directamente en Home Assistant, modifica la formación de los nombres de las entidades e incluye una larga lista de cambios incompatibles con versiones anteriores, entre ellos cambios en los rastreadores de dispositivos de FRITZ!Box Tools y en la autenticación mediante direcciones no locales. Según la publicación del blog de la versión, se añaden 38 integraciones nuevas. Las notas de la versión no mencionan ninguna corrección de seguridad con CVE. La versión está dirigida a todos los operadores de Home Assistant, especialmente a instalaciones con Custom Integrations de HACS, como las utilizadas en los artículos sobre [Midea PortaSplit](/blog/midea-portasplit-home-assistant) y [digitalSTROM](/blog/digitalstrom-home-assistant).

## Las versiones de un vistazo

| Versión | Fecha | Tipo | Puntos clave |
|---|---|---|---|
| 2026.11.0 | 04.11.2026 | Versión mensual | HACS pasa a ser Marketplace, nuevos nombres de entidades, los Device Trackers de los escáneres de red siempre con dispositivo, comprobación de autenticación en direcciones no locales, desaparece la carpeta `deps`, 38 integraciones nuevas |

## Marketplace: HACS pasa a formar parte de Home Assistant

Con la versión 2026.11, HACS pasa a formar parte de Home Assistant y se llama Marketplace, según la publicación del blog de la versión. Los administradores lo encontrarán en Ajustes > Marketplace e instalarán allí integraciones, tarjetas de panel, temas y plantillas; no se necesita una cuenta de GitHub. Tras instalar una integración, ya no es necesario reiniciar, aunque las actualizaciones siguen requiriendo un reinicio.

Home Assistant migra las instalaciones existentes de HACS durante el primer inicio tras la actualización. Los proyectos instalados permanecen instalados y continúan recibiendo actualizaciones; también se migran los repositorios personalizados y las entidades de actualización. Según el fabricante, no es necesaria ninguna intervención. El instalador de HACS finaliza con Home Assistant 2026.11, ya no es posible una instalación independiente de HACS en esta versión y el HACS anterior no recibirá nuevas versiones. En adelante, informe de los errores en el rastreador de incidencias de Home Assistant.

Al acceder por primera vez, Marketplace muestra un aviso indicando que los proyectos de la comunidad no son probados, revisados ni compatibles por parte del proyecto Home Assistant. Debe confirmar este aviso antes de instalar o actualizar. Los proyectos fuera del catálogo se pueden añadir mediante «Add from link»; para ello se necesita una conexión a GitHub mediante Device Sign-in, que solo lee información pública.

Para Midea PortaSplit, esto afecta a la Custom Integration `Midea AC LAN` y a la tarjeta `apexcharts-card`, siempre que ambas se hayan instalado mediante HACS. Según la información sobre la migración, permanecen instaladas y continúan recibiendo actualizaciones. El riesgo de cadena de suministro de las Custom Integrations, descrito en el artículo sobre [la protección de PortaSplit](/blog/midea-portasplit-home-assistant-absichern), se mantiene con Marketplace, como muestra el aviso de riesgo del fabricante. Tras la actualización, compruebe en Ajustes > Marketplace si están presentes todos los proyectos y repositorios personalizados que instaló anteriormente mediante HACS.

## Nuevos nombres de entidades

A partir de la versión 2026.11, los nombres de las entidades se componen del nombre de la entidad y de su contexto, normalmente el nombre del dispositivo. Los nombres asignados manualmente se migran según las mismas reglas y el nombre anterior se conserva como alias. Las tarjetas, insignias y filas sin nombre propio muestran ahora el nombre del dispositivo seguido del nombre de la entidad, en lugar del Friendly Name. Según las notas de la versión, los ID de entidad, las automatizaciones y los scripts no se ven afectados.

Para las plantillas, el fabricante recomienda las funciones `entity_name()`, `device_name()` y `area_name()`; el atributo `friendly_name` se marcará más adelante como obsoleto. Tras la actualización, revise los rótulos modificados en sus paneles y adapte las plantillas que evalúen `friendly_name`.

## FRITZ!Box Tools y Device Tracker

Los rastreadores de escáneres de red —entre los que se mencionan expresamente FRITZ!Box Tools, UniFi Network, NETGEAR y Nmap Tracker— ahora siempre tienen un dispositivo. Las entidades de rastreador renombradas reciben el nombre del dispositivo al principio de su nombre. Los rastreadores nuevos de estas integraciones siempre se crean desactivados; antes estaban activos si otra integración ya conocía la dirección MAC del dispositivo. Los rastreadores existentes conservan su estado, activado o desactivado.

Para las automatizaciones de presencia mediante la FRITZ!Box, como se describe para la salida automática en el [artículo sobre digitalSTROM](/blog/digitalstrom-home-assistant), esto significa que los rastreadores existentes siguen funcionando. Sin embargo, primero deberá activar el rastreador de un smartphone recién añadido antes de que proporcione un estado en las automatizaciones.

El changelog también enumera varias correcciones para FRITZ!Box Tools, entre ellas para la creación de los interruptores WLAN, el filtrado de hosts de FRITZ!Box sin dirección IP de los Device Trackers, un bucle de recarga en dispositivos sin datos de tráfico y un mensaje de error más comprensible en caso de autenticación no válida en el diálogo de configuración. La integración FRITZ!Box Call Monitor ahora informa de un error de configuración cuando faltan permisos.

## Midea y digitalSTROM

Las mejoras de Midea en la publicación del blog de la versión afectan a la integración oficial `Midea`, que forma parte de Core desde Home Assistant 2026.8. La Custom Integration `Midea AC LAN` (dominio `midea_ac_lan`) es independiente de ella. Se añaden controles y un sensor de progreso para lavadoras y secadoras, sensores de puerta para frigoríficos, entidades de calentadores de agua, un interruptor de encendido/apagado para purificadores de aire y un interruptor para el tono de señal de los deshumidificadores. Para los equipos de aire acondicionado, el changelog menciona un botón `reset_filter`, con el que se puede marcar el filtro como limpiado. La integración ha alcanzado el nivel de calidad Platinum. Según la página de la integración, el control y las consultas de estado funcionan localmente; la nube solo es necesaria para el token y la clave de dispositivos V3.

El changelog de 2026.11.0 no contiene ninguna entrada sobre digitalSTROM. La integración descrita en el [artículo sobre digitalSTROM](/blog/digitalstrom-home-assistant) es una Custom Integration; si está integrada mediante HACS como repositorio personalizado, se le aplica la migración al Marketplace descrita anteriormente.

## Otros cambios incompatibles con versiones anteriores

La lista completa se encuentra en la publicación del blog de la versión. Para el funcionamiento, son especialmente relevantes los siguientes puntos:

- **Auth:** Al acceder mediante una dirección IP no local, el esquema, el host y el puerto del navegador deben coincidir con la dirección que ve Home Assistant; de lo contrario, la autenticación falla con «Invalid client id». La comprobación anterior aceptaba estas direcciones por un error cuando contenían un puerto. En este caso, introduzca su proxy inverso en los Trusted Proxies de la integración HTTP o utilice un nombre de dominio.
- **Dependencias de Python:** Home Assistant ya no instala paquetes en la carpeta `deps` del directorio de configuración. Solo se ven afectadas las instalaciones Core fuera de un entorno virtual; no las instalaciones OS, Supervised ni de contenedor. Tras la actualización, se puede eliminar la carpeta `deps`.
- **SMTP:** Un adjunto no legible ahora provoca un error; antes, el mensaje se enviaba sin el adjunto.
- **Telegram:** Se elimina la antigua plataforma `notify`. Utilice las entidades Notify del bot de Telegram y la acción `notify.send_message`.
- **Z-Wave:** Se requieren zwave-js-server 3.11.0 o posterior (esquema 51), la aplicación Z-Wave JS a partir de la versión 1.9.0 o Z-Wave JS UI a partir de la versión v11.24.2.
- **Uptime Kuma:** Ya no se admiten versiones anteriores a 2.0.0.
- **Philips Hue:** Se elimina el efecto de luz `"None"`; se aplica en su lugar `"off"`.
- **Matter y Electra Smart:** `climate.turn_on` continúa con el último modo activo. Para un modo específico, utilice `climate.set_hvac_mode`.
- **Intervalo de sondeo:** En Hayward Omnilogic, Hive, Keenetic NDMS2, Kraken, Plaato y Risco se ignora un intervalo propio y se utiliza el predeterminado. Control4 consulta siempre cada 5 segundos; para usar otro intervalo, desactive el sondeo y llame a `homeassistant.update_entity` desde una automatización.

La integración Flipr se ha eliminado porque la empresa responsable fue liquidada.

## Nuevas funciones

Además de Marketplace, la publicación del blog de la versión menciona, entre otras cosas, la exclusión de entidades, dispositivos, áreas, plantas o etiquetas individuales de un destino, copiar y pegar secciones de paneles, una tarjeta de fecha y una tarjeta de alerta con editor visual. El panel de energía puede derivar un sensor de energía a partir de un sensor de potencia. Para la resolución de problemas, hay un filtro «Triggered by» en la actividad, registros en directo por integración y eliminación de estadísticas en un periodo determinado. Las copias de seguridad y los medios se pueden guardar en unidades locales. Se incorporan compatibilidad con cámaras Matter 1.5 con WebRTC, así como con LoRaWAN (ChirpStack, The Things Network, SenseCAP) y Zigbee Green Power.

En la tarjeta de alerta, la opción `color` ahora se denomina `alert_color`; solo debe adaptar los paneles que haya creado usted mismo en YAML. La nueva integración Debug console ejecuta código Python dentro de Home Assistant, tiene acceso completo y, por ello, debe activarse en `configuration.yaml`; solo pueden utilizarla los administradores.

## Procedimiento recomendado

La versión objetivo es la 2026.11.0. Antes de actualizar, lea los cambios incompatibles con versiones anteriores en la publicación del blog de la versión para las integraciones que utilice. Después de la actualización, compruebe en Marketplace si se han migrado todos los proyectos HACS anteriores, revise los nuevos nombres de entidades en los paneles y las plantillas y, si es necesario, active los Device Trackers creados recientemente. Si accede mediante un proxy inverso, pruebe la autenticación. Los avisos de seguridad del repositorio de GitHub de Home Assistant Core no contienen, a 5 de noviembre de 2026, ninguna entrada asignada a 2026.11.0; la publicación del blog de la versión no incluye información sobre Known Issues.

## Fuentes

1.  [Home Assistant: 2026.11: A lot in store!](https://www.home-assistant.io/blog/2026/11/04/release-202611/): Publicación del blog de la versión con Marketplace, nombres de entidades, nuevas integraciones y cambios incompatibles con versiones anteriores.

2.  [Home Assistant: Full changelog for Home Assistant 2026.11](https://www.home-assistant.io/changelogs/core-2026.11): Changelog completo, incluidas las entradas sobre FRITZ!Box Tools, FRITZ!Box Call Monitor y Midea.

3.  [GitHub: home-assistant/core Release 2026.11.0](https://github.com/home-assistant/core/releases/tag/2026.11.0): Etiqueta de versión publicada el 4 de noviembre de 2026.

4.  [Home Assistant: Midea](https://www.home-assistant.io/integrations/midea/): Documentación de la integración oficial de Midea, introducida en 2026.8, con control local.

5.  [GitHub: Security Advisories home-assistant/core](https://github.com/home-assistant/core/security): Avisos de seguridad publicados, a 5 de noviembre de 2026 sin entrada sobre 2026.11.0.

6.  [Midea PortaSplit in Home Assistant: Einrichtung und Dashboard](/blog/midea-portasplit-home-assistant): Instalación de Midea AC LAN y apexcharts-card mediante HACS.

7.  [Midea PortaSplit in Home Assistant absichern](/blog/midea-portasplit-home-assistant-absichern): Riesgo de cadena de suministro de las Custom Integrations.

8.  [digitalSTROM mit Home Assistant verbinden](/blog/digitalstrom-home-assistant): Integración local de digitalSTROM y presencia mediante la FRITZ!Box.
