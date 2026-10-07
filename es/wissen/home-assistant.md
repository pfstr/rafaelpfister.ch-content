---
title: "Home Assistant: arquitectura, modelo de datos y operación"
blatt: "home-assistant"
description: "Home Assistant para administradores de plataformas, redes e IoT: Core de Python orientado a eventos, integraciones y registros, Home Assistant OS y contenedores, puentes de protocolos y radiofrecuencia, tiempo de ejecución de automatizaciones, Recorder, API, autenticación, observabilidad, copia de seguridad y recuperación."
fakten:
  - label: Función del sistema
    wert: plataforma central de control y automatización orientada a eventos para dispositivos locales, redes de radio y servicios externos
    href: https://developers.home-assistant.io/docs/architecture_index/
  - label: Core
    wert: Event Bus, State Machine, Service Registry y Timer constituyen el núcleo de ejecución
    href: https://developers.home-assistant.io/docs/architecture/core/
  - label: Pila tecnológica
    wert: Home Assistant Core y sus integraciones están implementados en Python; asyncio proporciona el procesamiento de E/S concurrente
    href: https://github.com/home-assistant/core
  - label: Modelo de extensiones
    wert: las integraciones constan de lógica de dominio y plataformas; los Config Entries controlan su ciclo de vida persistente
    href: https://developers.home-assistant.io/docs/architecture_components/
  - label: Modelo de objetos
    wert: Config Entry → dispositivo → entidad → estado; los registros estabilizan identidades, nombres y asignaciones
    href: https://developers.home-assistant.io/docs/architecture/devices-and-services/
  - label: Instalación compatible
    wert: Home Assistant OS como appliance gestionado o Home Assistant Container en un host operado por el propio administrador
    href: https://www.home-assistant.io/faq/ha-vs-hassio/
  - label: Pila de HAOS
    wert: Buildroot, Linux, systemd, Docker, Supervisor, Core y apps; RAUC actualiza el sistema operativo
    href: https://developers.home-assistant.io/docs/operating-system/
  - label: Interfaces
    wert: REST mediante /api y WebSocket mediante /api/websocket en el mismo extremo HTTP que el frontend
    href: https://developers.home-assistant.io/docs/api/rest/
  - label: Puerto predeterminado
    wert: TCP 8123 para frontend, REST y WebSocket; TLS o un proxy inverso modifican la ruta de acceso externa
    href: https://www.home-assistant.io/integrations/http/
  - label: Historial
    wert: Recorder escribe estados y eventos seleccionados mediante SQLAlchemy en SQLite de forma predeterminada; MariaDB, MySQL y PostgreSQL son compatibles
    href: https://www.home-assistant.io/integrations/recorder/
  - label: Automatizaciones
    wert: los triggers inician una ejecución, las conditions deciden y las actions usan la misma semántica secuencial que los scripts
    href: https://www.home-assistant.io/docs/automation/basics/
  - label: Recuperación
    wert: las copias de seguridad cifradas pueden restaurar la configuración, Core y apps; las claves, los controladores de radio y las bases de datos externas siguen siendo dependencias independientes
    href: https://www.home-assistant.io/common-tasks/general/
werbung:
  - newsletter
ctaThemen:
  - smart-home-iot
translationSourceHash: 204801ccce2af55eaa473c7a7599bd0744a8e9db9b0e1dbdb99efe8949a6fff1
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T10:54:14.894Z
translationReview: automatic
---

# Home Assistant: arquitectura, modelo de datos y operación

Home Assistant es una plataforma central de control y automatización para dispositivos, redes de radio, servicios IP e interfaces de usuario. La instancia recopila estados mediante integraciones, los normaliza como entidades, distribuye los cambios a través de un Event Bus y ejecuta acciones a partir de ellos. «Local» designa una preferencia arquitectónica, no una característica general de cada integración: una bombilla Zigbee puede ser plenamente accesible de forma local, mientras que una integración del fabricante obtiene sus estados exclusivamente de una API en la nube. La [visión general de arquitectura oficial](https://developers.home-assistant.io/docs/architecture_index/) separa el sistema operativo, Supervisor y Core; la [arquitectura de integraciones](https://developers.home-assistant.io/docs/architecture_components/) describe la ampliación de Core mediante componentes de Python.

Para administradores, Home Assistant no es simplemente un panel ni un convertidor universal de protocolos. Es un orquestador con estado y varios posibles puntos de fallo: tiempo de ejecución de Python, integraciones, registros, base de datos, autenticación, redes locales, controladores de radio, brokers, nubes de fabricantes y, en su caso, aplicaciones de Supervisor. Una interfaz en verde solo prueba que funciona la ruta del frontend. No prueba que los eventos lleguen a tiempo, que los dispositivos sean accesibles, que las automatizaciones se ejecuten de forma determinista ni que una copia de seguridad sea restaurable, incluidas las dependencias externas.

La explicación sigue un evento de dispositivo a través de la integración, Event Bus y State Machine hasta la automatización y la acción. Después se sitúan la persistencia, los complementos, la seguridad, la supervisión y la recuperación.

## Enfoque arquitectónico: nodo central de eventos y estados

Home Assistant Core está orientado a eventos. Cuatro componentes documentados forman el núcleo ([arquitectura de Core](https://developers.home-assistant.io/docs/architecture/core/)):

1. El **Event Bus** distribuye eventos a listeners registrados.
2. La **State Machine** mantiene el último estado conocido de cada entidad cargada y publica `state_changed`.
3. El **Service Registry** administra acciones invocables y procesa llamadas a servicios.
4. El **Timer** genera eventos temporales para el procesamiento dependiente del tiempo.

Las integraciones traducen estados de dispositivos o servicios a este modelo. Una integración puede realizar polling, recibir eventos push, utilizar bibliotecas locales o consultar una API remota. Home Assistant unifica el estado resultante, no el transporte. Este es el límite operativo más importante: dos entidades con el mismo tipo de dominio, por ejemplo `light`, pueden tener rutas de latencia, autenticación y recuperación completamente distintas.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1116" src="/images/kb-interaktiv-home-assistant.svg?v=20260813" title="Interaktive Infografik: Home Assistant von Geräten und Protokollbrücken über Integrationen, Registries, Event Bus, State Machine, Automationen und Recorder bis zu APIs, Supervisor, Monitoring und Recovery" loading="lazy">
  <a href="/images/kb-interaktiv-home-assistant.svg?v=20260813">Abrir directamente el gráfico interactivo</a>.
</iframe>

## Capas de ejecución y modelos de instalación

Home Assistant ofrece dos modelos de instalación compatibles. **Home Assistant OS** es un appliance gestionado. **Home Assistant Container** ejecuta Home Assistant Core como contenedor en un host bajo responsabilidad del operador. La comparación oficial recomienda HAOS para casi todas las instalaciones y describe Container como una instalación independiente de Core sin aplicaciones de Supervisor ([HAOS o Container](https://www.home-assistant.io/faq/ha-vs-hassio/)).

### Home Assistant OS

HAOS se crea con Buildroot y consta de Linux, GNU C Library, systemd y Docker. SquashFS alberga las áreas del sistema de solo lectura, ZRAM los sistemas de archivos temporales y swap, AppArmor limita procesos y RAUC actualiza el sistema operativo ([Home Assistant Operating System](https://developers.home-assistant.io/docs/operating-system/)). Por encima, el **Supervisor** gestiona Core, apps, DNS, audio, mDNS, copias de seguridad y actualizaciones ([Supervisor](https://developers.home-assistant.io/docs/supervisor/)).

El modelo de appliance reduce variantes, pero transfiere al Supervisor una responsabilidad amplia. Un fallo puede estar en al menos cinco niveles: ranura de arranque/SO, Docker Engine, Supervisor, contenedor de Core o una app individual. El Supervisor puede revertir una ruta fallida de actualización de Core; este mecanismo no detecta automáticamente un comportamiento funcionalmente erróneo de dispositivos o bases de datos.

### Home Assistant Container

Container proporciona únicamente Core. El sistema operativo anfitrión, el motor de contenedores, la red, los volúmenes, la base de datos, el broker, el servidor de radio, el proxy inverso, la copia de seguridad y las actualizaciones pertenecen al operador. Las apps de Supervisor son servicios empaquetados por separado. En un diseño de contenedores, Mosquitto, Matter Server, Zigbee2MQTT, Z-Wave JS UI, PostgreSQL o un proxy inverso se operan como cargas de trabajo independientes con sus propios volúmenes, versiones y health checks.

La ventaja es una arquitectura de plataforma explícita; el precio es una mayor superficie operativa. Una copia de seguridad del volumen de Core no incluye, por ejemplo, la base de datos externa de Recorder, el estado del broker, la NVM de radio ni las claves del proxy inverso si se encuentran fuera.

### Formas históricas de instalación

La anterior instalación **Core** en un entorno Python y la instalación **Supervised** en un Linux autogestionado se discontinuaron en 2025. Desde la versión 2025.12 se consideran no compatibles; las arquitecturas de 32 bits `i386`, `armhf` y `armv7` perdieron al mismo tiempo la ruta de lanzamiento. El anuncio del proyecto nombra HAOS y Container como los modelos restantes ([discontinuación de Core y Supervised](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/)). Por ello, un nombre histórico de instalación no debe confundirse con el componente de software **Home Assistant Core**, que sigue ejecutándose también dentro de HAOS y Container.

## Pila tecnológica

Home Assistant Core es una aplicación Python bajo licencia Apache-2.0. El [repositorio oficial de Core](https://github.com/home-assistant/core) muestra Python, asyncio y la estructura modular de integraciones. Las integraciones intensivas en E/S no deben bloquearse: las reglas de calidad prefieren dependencias asíncronas para que las llamadas de red y dispositivos no detengan el Event Loop compartido ([dependencia asíncrona](https://developers.home-assistant.io/docs/core/integration-quality-scale/rules/async-dependency/)). El código de bibliotecas bloqueante se desplaza a hilos de executor; no obstante, el trabajo intensivo de CPU o mal acotado sigue siendo un riesgo de capacidad y latencia.

La pila visible abarca más que Python:

| Capa | Tecnología típica | Relevancia operativa |
|---|---|---|
| Frontend | Aplicación de navegador, HTTP y WebSocket | Ruta de usuario y tiempo real |
| Core | Python, asyncio, integraciones | Estados, eventos, acciones, autenticación |
| Persistencia | Almacenes de configuración basados en JSON, YAML, SQLAlchemy/SQL | Configuración, registros, historial |
| HAOS | Buildroot, Linux, systemd, Docker, AppArmor, RAUC | Ciclo de vida e aislamiento del appliance |
| Servicios | Apps de Supervisor o contenedores/hosts externos | MQTT, Matter, base de datos, proxy, comparticiones de archivos |
| Edge | Controladores de radio, puentes de protocolos, API de dispositivos y nube | Accesibilidad física y procedencia de datos |

La [Integration Quality Scale](https://developers.home-assistant.io/docs/core/integration-quality-scale/) evalúa las integraciones según flujo de configuración, pruebas, tipado, diagnóstico, uso eficiente de datos y comportamiento asíncrono. Un nivel elevado mejora la mantenibilidad previsible, pero no es un SLA de disponibilidad para el dispositivo subyacente ni para el proveedor de nube.

Tras el modelo de instalación y el tiempo de ejecución, sigue el modelo de datos. Solo si se distinguen Config Entry, dispositivo, entidad y estado pueden explicarse adecuadamente las entidades duplicadas, dispositivos ausentes y automatizaciones defectuosas.

## Modelo de objetos: Config Entry, dispositivo, entidad y estado

La clave operativa del inventario no es el mosaico visible, sino la cadena de configuración, identidad de dispositivo y entidad.

### Config Entries

Un **Config Entry** almacena la configuración persistente de una instancia de integración. Un flujo de configuración de UI lo crea; opciones, reconfiguración, recarga, descarga, eliminación y migración son operaciones definidas de ciclo de vida. Las integraciones no deben mutar directamente los datos de Entry, sino utilizar el administrador de Config Entry ([Config entries](https://developers.home-assistant.io/docs/config_entries_index/)). Por tanto, un error de autenticación, un Entry no cargado y un extremo inaccesible son estados diferentes.

### Dispositivos y registros

El **Device Registry** agrupa extremos técnicos en dispositivos. Identificadores o conexiones, por ejemplo número de serie y dirección MAC, sirven para la correspondencia; `via_device` puede representar una relación de puente o principal ([Device registry](https://developers.home-assistant.io/docs/device_registry_index/)). De este modo, un sensor Zigbee puede aparecer como dispositivo conectado mediante un Coordinator, sin que el Coordinator sea su estado de aplicación.

El **Entity Registry** asigna a las entidades una identidad permanente con `unique_id` y evita Entity IDs en conflicto. La dirección IP, el nombre de host, la URL, el nombre de usuario o la dirección de correo electrónico no se consideran explícitamente Unique IDs estables ([Entity registry](https://developers.home-assistant.io/docs/entity_registry_index/)). Esto explica por qué renombrar manualmente un host no puede sustituir la identidad de dispositivo y por qué las migraciones de integraciones requieren identificadores estables del fabricante.

### Entidad y estado

Una **entidad** representa una función o una magnitud medida: `sensor`, `switch`, `light`, `climate`, `binary_sensor` u otro dominio. Su estado consta de un State principal, atributos, horas de cambio y Context. La State Machine solo mantiene el último estado conocido. `unavailable` significa que la entidad no está siendo atendida actualmente por un objeto Entity activo; `unknown` significa que no hay un valor utilizable. Por tanto, «último valor» no equivale automáticamente a «medición reciente».

La [interacción documentada de dispositivos y servicios](https://developers.home-assistant.io/docs/architecture/devices-and-services/) distingue Entity Integration, Entity Component, Entity Platform e integración específica del fabricante. Por ello, en el diagnóstico siempre se pregunta:

- ¿Qué Config Entry posee la entidad?
- ¿Mediante qué integración y plataforma se crea?
- ¿Qué Device ID y Entity ID estables vinculan historial y configuración?
- ¿Se realiza polling o push?
- ¿Qué modelo temporal y de disponibilidad tiene el valor de origen?
- ¿Qué puente, biblioteca, API en la nube o enlace de radio hay delante?

## Integraciones y aislamiento de fallos

Una integración define un dominio y puede proporcionar plataformas como `sensor`, `light` o `switch`. La plataforma abstrae el tipo de entidad; la integración de dispositivo utiliza el protocolo concreto. Las integraciones integradas se distribuyen con Core y se prueban mediante su proceso de lanzamiento. Sin embargo, las **Custom Integrations** se ejecutan en el mismo proceso de Python y pueden afectar a imports, Event Loop, tiempo de arranque o consumo de memoria. Por ello, el directorio `/config/custom_components` forma parte del inventario, la gestión de cambios y la recuperación.

La [visión general oficial de integraciones](https://www.home-assistant.io/integrations/) distingue, entre otras, clases de IoT como Local Push, Local Polling, Cloud Push y Cloud Polling. Esta clasificación es más útil para los modelos operativos que una larga lista de fabricantes:

| Clase | Ruta de datos | Área típica de fallo |
|---|---|---|
| Local Push | El dispositivo o puente envía a la LAN | Multicast, firewall, puente, subred |
| Local Polling | Core consulta el dispositivo local | Latencia, timeout, intervalo de consulta, capacidad del dispositivo |
| Cloud Push | La nube envía o transmite eventos | Internet, cuenta, token, stream del proveedor |
| Cloud Polling | Core consulta la API del proveedor | Rate limit, token, Internet, cambios de API |
| Calculated/Internal | Core calcula el estado | Datos de entrada, plantillas, hora, estado tras reinicio |

La integración no es un aislador de procesos. Una delimitación limpia de fallos desactiva o recarga específicamente el Config Entry afectado antes de reiniciar todo Core. Un reinicio destruye evidencia volátil y puede restablecer temporizadores de automatizaciones dependientes del tiempo.

## Modelo de protocolos y red

Para Home Assistant, un **grafo de dependencias** es más útil que una tabla OSI general. La plataforma se sitúa en la capa de aplicación, pero sus rutas de datos se bifurcan:

- Frontend, REST y WebSocket se ejecutan sobre HTTP en TCP, de forma predeterminada en el puerto 8123.
- DNS resuelve hosts y servicios en la nube; mDNS y SSDP descubren dispositivos en la red local.
- MQTT utiliza un broker independiente y un modelo publish/subscribe sobre TCP o WebSocket.
- Zigbee, Z-Wave, Thread y Bluetooth requieren controladores de radio o proxies de red.
- Matter utiliza comunicación IP, pero para el aprovisionamiento y la operación de Fabric requiere un servidor Matter y, en su caso, un Thread Border Router.
- Las integraciones de fabricantes pueden utilizar HTTPS, protocolos locales propietarios o streams en la nube.

Las integraciones de descubrimiento integradas documentan [mDNS/Zeroconf](https://www.home-assistant.io/integrations/zeroconf/) y [SSDP](https://www.home-assistant.io/integrations/ssdp/). Ambas dependen del segmento y multicast. Un proxy inverso para el frontend no corrige el descubrimiento a través de límites de VLAN. Multicast relay, IGMP snooping, aislamiento de clientes Wi-Fi, IPv6-RA, sufijos DNS y reglas de firewall se verifican para cada ruta real de dispositivo.

### MQTT como espacio de estados propio

MQTT no es el Event Bus interno. Es un servicio de broker externo con el que se comunica una integración. La [integración MQTT oficial](https://www.home-assistant.io/integrations/mqtt/) describe Discovery Topics, retained messages, Birth/Last Will, Availability, TLS y MQTT 5. La Discovery retenida puede volver a crear dispositivos después de un reinicio, pero también puede conservar Ghost Entities obsoletas. La disponibilidad requiere una semántica propia; un State retenido existente no prueba que el publicador siga activo.

Una operación MQTT sólida inventaría el broker, Client IDs, autenticación, CA, Topics, QoS, Retain, Expiry, Birth/Will y el origen de Discovery. La copia de seguridad del broker y la de Core son objetos de protección separados.

### Rutas de radio y puentes

[ZHA](https://www.home-assistant.io/integrations/zha/) integra un Zigbee Coordinator, [Z-Wave JS](https://www.home-assistant.io/integrations/zwave_js/) utiliza un servidor Z-Wave JS independiente y [Matter](https://www.home-assistant.io/integrations/matter/) conecta un servidor Matter. [Thread](https://www.home-assistant.io/integrations/thread/) administra referencias a Border Router y redes, pero no es idéntico a Matter. Los dispositivos de radio, el firmware del controlador, los datos de red, el material de claves y la configuración de dispositivos forman respectivamente un conjunto de recuperación. Mover una memoria USB o sustituir un Coordinator no es un cambio ordinario de dirección IP.

Las integraciones proporcionan estados y eventos; las automatizaciones reaccionan a ellos. Por ello, su secuencia de Trigger, Conditions y Actions debe diagnosticarse separadamente de la configuración de dispositivos.

## Tiempo de ejecución de automatizaciones: Trigger, Condition, Action

Una automatización es una definición de ejecución reactiva. Los [fundamentos de automatizaciones](https://www.home-assistant.io/docs/automation/basics/) separan Trigger, Conditions opcionales y Actions. El Trigger genera una ejecución, las Conditions comprueban el estado de entrada y las Actions utilizan la semántica de secuencia de los Scripts ([Actions](https://www.home-assistant.io/docs/automation/action/)).

El momento es importante: State, atributos y valores de plantilla pueden cambiar entre el Trigger y una Action posterior. Un Delay no mantiene abierta una transacción. Por ello, varias ejecuciones de la misma automatización requieren un modo como Single, Restart, Queued o Parallel y un modelo de conflictos consciente. Los actuadores físicos rara vez son transaccionales; una ejecución parcialmente realizada puede requerir Actions de compensación.

La [documentación de Trigger](https://www.home-assistant.io/docs/automation/trigger/) indica que las esperas `for` no sobreviven a un reinicio ni a una recarga de automatizaciones. Quien deba mantener un plazo entre reinicios persiste un instante, por ejemplo en `input_datetime`, y lo activa contra este. Las Conditions son solo comprobaciones en la ejecución actual; la [semántica de Condition](https://www.home-assistant.io/docs/scripts/conditions/) no las convierte en un bloqueo contra cambios paralelos.

Las plantillas se evalúan en Home Assistant con expresiones Jinja. Los errores de entrada y tipo, `unknown`, `unavailable`, zonas horarias y conversión implícita de cadenas deben incluirse en las pruebas. La [documentación de templating](https://www.home-assistant.io/docs/automation/templating/) describe variables dependientes de Trigger. Un administrador no prueba solo el Happy Path, sino también reinicio, entidad ausente, evento tardío, Trigger duplicado y error de actuador.

## Configuración, registros y fuente de verdad

Home Assistant combina Config Entries guiados por UI, datos de registro y YAML. `configuration.yaml` es la raíz de la configuración manual, pero no la Source of Truth completa. La [visión general oficial de configuración](https://www.home-assistant.io/docs/configuration/) distingue UI y YAML; los Packages pueden estructurar bloques YAML relacionados ([Packages](https://www.home-assistant.io/docs/configuration/packages/)).

Para Git y revisión solo es adecuado el componente textual sin secretos. `secrets.yaml` separa valores de YAML, pero no los cifra; la [guía de endurecimiento](https://www.home-assistant.io/docs/configuration/securing/) lo indica expresamente. El estado de UI, registros, tokens y Config Entries se encuentran en el almacenamiento de configuración y se modifican mediante vías de UI/API compatibles. Editar directamente archivos de almacenamiento interno mientras Core está en ejecución omite la lógica de esquema, ciclo de vida y consistencia.

Un inventario de configuración incluye:

- YAML, Packages, Blueprints y Custom Components,
- Config Entries junto con origen, propietario y autenticación,
- asignaciones de Device, Entity y Area,
- automatizaciones, Scripts, escenas y Dashboards,
- usuarios, tokens, MFA y proveedores de identidad externos,
- apps de Supervisor o servicios externos,
- controladores de radio, broker, base de datos y proxy,
- secretos, certificados y claves de recuperación.

## Recorder, historial y estadísticas a largo plazo

La State Machine mantiene el estado actual en memoria. El historial solo se crea mediante **Recorder**. Escribe cambios de estado y eventos seleccionados mediante SQLAlchemy en una base de datos; History, Activity, gráficos y estadísticas a largo plazo se leen desde ella. La [documentación oficial de Recorder](https://www.home-assistant.io/integrations/recorder/) nombra SQLite como predeterminado y recomendado, además de MariaDB, MySQL y PostgreSQL como alternativas compatibles.

Los datos de Recorder no son una fuente de eventos para el control en tiempo real. Una base de datos caída puede afectar al historial y las estadísticas mientras los estados actuales y las automatizaciones continúan parcialmente. A la inversa, un historial completo no prueba que una Action se haya realizado correctamente en el dispositivo físico.

Los parámetros operativos principales son:

- `purge_keep_days` para historial sin procesar,
- filtros Include/Exclude para entidades y eventos,
- `commit_interval` como relación entre E/S y ventana de pérdida,
- tamaño de la base de datos, espacio libre y latencia de escritura,
- Purge y Repack,
- orden de inicio y accesibilidad de bases de datos externas,
- estadísticas a largo plazo y coherencia de metadatos.

Un cambio de base de datos de Recorder no migra el historial existente de forma compatible. Las bases de datos externas necesitan sus propias copias de seguridad coherentes y pruebas de restauración. Para SQLite, la documentación indica espacio libre de al menos 2,5 veces el tamaño de la base de datos para el tratamiento de corrupción. Por tanto, Storage y Recorder constituyen una ruta propia de capacidad y recuperación, no solo una caché opcional.

## API, WebSocket y autenticación

El frontend y las API comparten de forma predeterminada el mismo listener HTTP. La [REST API](https://developers.home-assistant.io/docs/api/rest/) utiliza JSON y Bearer Tokens; la ruta base es `/api/`. La [WebSocket API](https://developers.home-assistant.io/docs/api/websocket/) está en `/api/websocket`, pasa por `auth_required`, `auth` y `auth_ok`, y correlaciona comandos mediante ID numéricos. WebSocket proporciona streams de eventos y registros de forma más eficiente que el polling REST repetido.

Los tokens de larga duración son credenciales de usuario. La [Authentication API](https://developers.home-assistant.io/docs/auth_api/) describe OAuth/IndieAuth, Refresh Tokens, Long-Lived Access Tokens y Signed Paths de corta duración. Un token hereda el contexto de su usuario; un Long-Lived Token válido durante diez años debe estar en un almacén de secretos, no en YAML, historial de shell, URL o JavaScript de Dashboard.

Un monitor de API comprueba al menos autenticación, `/api/config`, entidades esperadas, `last_updated`, suscripción WebSocket y una ruta segura de lectura/acción. Un HTTP 200 en `/` solo comprueba la accesibilidad del frontend.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Home-Assistant-API-Inventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$headers = @{ Authorization = "Bearer $env:HA_TOKEN" }
Invoke-RestMethod -Headers $headers -Uri "https://ha.example.net/api/config"
Invoke-RestMethod -Headers $headers -Uri "https://ha.example.net/api/states/sensor.uptime" |
  ConvertTo-Json -Depth 8</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --fail --silent --show-error \
  -H "Authorization: Bearer $HA_TOKEN" \
  https://ha.example.net/api/config | jq .
curl --fail --silent --show-error \
  -H "Authorization: Bearer $HA_TOKEN" \
  https://ha.example.net/api/states/sensor.uptime | jq .</code></pre>
  </div>
</div>

[`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) y [`ConvertTo-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json) procesan la consulta de Windows; [`curl`](https://curl.se/docs/manpage.html) y [`jq`](https://jqlang.org/manual/) hacen lo mismo en Unix. El token solo se muestra como variable de entorno del proceso; en producción procede de un almacén de secretos controlado.

## HTTP, TLS y proxy inverso

El extremo HTTP escucha de forma predeterminada en TCP 8123. TLS directo, proxy inverso y Home Assistant Cloud son modelos de acceso diferentes. Con un proxy inverso tradicional, `use_x_forwarded_for` y `trusted_proxies` deben configurarse adecuadamente; de lo contrario, la IP de cliente será incorrecta o la solicitud se rechazará ([HTTP integration](https://www.home-assistant.io/integrations/http/)). Una lista amplia de confianza de proxies permite falsificar información Forwarded-For.

La [guía de seguridad](https://www.home-assistant.io/docs/configuration/securing/) recomienda contraseñas únicas, MFA, privilegios mínimos de administrador y acceso remoto protegido en vez de exposición directa a Internet. TLS solo termina el transporte. Los permisos de token, encabezados de proxy, actualizaciones WebSocket, límites de tasa, DNS, renovación de certificados y la seguridad del IdP anterior siguen siendo controles independientes.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DNS-, TCP- und TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName ha.example.net
Test-NetConnection ha.example.net -Port 443
curl.exe -sS -D - -o NUL https://ha.example.net/api/
Get-NetTCPConnection -State Established | Where-Object RemotePort -eq 443</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig +short ha.example.net A ha.example.net AAAA
curl -sS -D - -o /dev/null https://ha.example.net/api/
ss -ntp state established '( dport = :443 )'
openssl s_client -connect ha.example.net:443 -servername ha.example.net -brief</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname), [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) y [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) comprueban Windows; [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility), [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) y [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) comprueban Unix. La llamada no autenticada `/api/` puede devolver 401; lo determinante son la resolución de nombres, la identidad TLS, la ruta de proxy y el límite de autenticación esperado.

## Diagnóstico MQTT

El estado del broker se comprueba fuera de Home Assistant. Un subscriber observa Discovery, Availability y State sin modificar los Topics. Una prueba de Publish utiliza una ruta de prueba reservada específicamente; los Command Topics de producción no se describen de paso.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für MQTT-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">mosquitto_sub.exe -h mqtt.example.net -p 8883 --cafile .\ca.pem `
  -u ha-observer -P $env:MQTT_PASSWORD -v -t "homeassistant/#"
mosquitto_pub.exe -h mqtt.example.net -p 8883 --cafile .\ca.pem `
  -u ha-probe -P $env:MQTT_PASSWORD -t "ops/probe" -m "online" -q 1</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">mosquitto_sub -h mqtt.example.net -p 8883 --cafile ./ca.pem \
  -u ha-observer -P "$MQTT_PASSWORD" -v -t 'homeassistant/#'
mosquitto_pub -h mqtt.example.net -p 8883 --cafile ./ca.pem \
  -u ha-probe -P "$MQTT_PASSWORD" -t 'ops/probe' -m 'online' -q 1</code></pre>
  </div>
</div>

[`mosquitto_sub`](https://mosquitto.org/man/mosquitto_sub-1.html) y [`mosquitto_pub`](https://mosquitto.org/man/mosquitto_pub-1.html) son los clientes oficiales del broker. Las contraseñas en la línea de comandos pueden ser visibles en listas de procesos o historial; los ejemplos ilustran la ruta, mientras que la llamada de producción utiliza un archivo de contraseñas, un almacén de secretos del sistema operativo o credenciales de corta duración.

## Operación de HAOS y Container

HAOS proporciona el comando `ha` mediante acceso de terminal/SSH. Las instalaciones de Container se operan con las herramientas del runtime elegido. Un paquete de diagnóstico reúne información del sistema, registro de Core, diagnóstico de integración, estado de contenedores, espacio libre y momento temporal.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Home-Assistant-Laufzeitdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">docker inspect homeassistant | ConvertFrom-Json
docker logs --since 30m --timestamps homeassistant 2&gt;&amp;1 |
  Select-String -Pattern 'ERROR|WARNING|unavailable|timeout'
docker stats --no-stream homeassistant</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">docker inspect homeassistant | jq '.[0].State, .[0].Mounts, .[0].NetworkSettings.Networks'
docker logs --since 30m --timestamps homeassistant 2&gt;&amp;1 | grep -E 'ERROR|WARNING|unavailable|timeout'
docker stats --no-stream homeassistant
df -h /path/to/config &amp;&amp; du -sh /path/to/config</code></pre>
  </div>
</div>

[`docker inspect`](https://docs.docker.com/reference/cli/docker/inspect/), [`docker logs`](https://docs.docker.com/reference/cli/docker/container/logs/) y [`docker stats`](https://docs.docker.com/reference/cli/docker/container/stats/) proporcionan el estado de contenedores. [`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json) y [`Select-String`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-string) procesan salidas de Windows; [`grep`](https://www.gnu.org/software/grep/manual/grep.html), [`df`](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html) y [`du`](https://www.gnu.org/software/coreutils/manual/html_node/du-invocation.html) complementan Unix. Un contenedor en ejecución es solo la primera comprobación; después siguen la ruta de integración, registro, evento y dispositivo.

Para solucionar problemas, la ruta de señal se lee hacia atrás: acción, Automation Trace, cambio de estado, integración, protocolo de red y dispositivo físico.

## Observabilidad y diagnóstico sistemático

**System Health** recopila tipo de instalación, arquitectura e información de Python, Core y frontend, y proporciona funciones de diagnóstico mediante Configuración > Sistema > Reparaciones ([System Health](https://www.home-assistant.io/integrations/system_health/)). La [integración Logger](https://www.home-assistant.io/integrations/logger/) controla los niveles de registro globales y específicos por componente. El registro de depuración se limita temporalmente y a los namespaces afectados; de lo contrario, las tormentas de radio o eventos pueden dominar memoria y E/S.

Una cadena de diagnóstico sólida es:

1. **Síntoma y estado esperado:** ¿Qué entidad, Action, automatización o interfaz está afectada?
2. **Tiempo y alcance:** ¿Desde cuándo, para qué dispositivos, usuarios, redes e instancias de integración?
3. **Identidad del objeto:** Asegurar Config Entry, Device ID, Entity ID, Unique ID y relación de puente.
4. **Tiempo de ejecución:** Comprobar Core, Event Loop, memoria, CPU, sistema de archivos y base de datos.
5. **Integración:** Comprobar estado de Entry, autenticación, estado de Coordinator/polling y descarga de diagnóstico.
6. **Transporte:** Comprobar Discovery, DNS, TCP, TLS, broker, controlador de radio o API del fabricante.
7. **Automatización:** Comprobar Trace, datos de Trigger, Conditions, Run Mode y resultado de Action.
8. **Persistencia:** Evaluar el retraso de Recorder y el historial por separado del estado en vivo.
9. **Prueba controlada:** Utilizar una entidad de prueba de solo lectura o inocua.
10. **Recuperación:** Recargar antes de reiniciar, reiniciar antes de restaurar; asegurar evidencia previamente.

Una entidad `unavailable` puede proceder de un Config Entry descargado, puente ausente, pérdida de radio o timeout de origen. Un número antiguo visible es más peligroso porque parece plausible. Por tanto, la supervisión necesita límites de frescura, no solo límites de valor.

## Actualizaciones, versiones y Custom Integrations

Home Assistant publica versiones frecuentes de Core y documenta cambios incompatibles con versiones anteriores. Un artículo de referencia estático no congela deliberadamente una versión actual. En su lugar, el despliegue comprueba en el momento de mantenimiento las notas de versión, los cambios de integración y las dependencias de destino.

Una ruta de actualización controlada incluye:

1. Confirmar la copia de seguridad y la descarga independiente o ubicación de almacenamiento externa.
2. Comprobar espacio libre, estado de base de datos y System Health.
3. Evaluar notas de versión, integraciones afectadas y Custom Components.
4. Inventariar dependencias de radio, broker, base de datos y proxy.
5. Actualizar Core o HAOS y apps en el orden definido.
6. Comprobar registro de inicio, reparaciones y migraciones de registros.
7. Probar rutas críticas de sensores, actuadores, automatizaciones, API y acceso remoto.
8. Determinar el límite de error y solo entonces iniciar rollback o restauración.

HAOS utiliza RAUC con dos ranuras de sistema operativo; `ha os info` y `rauc status` muestran el estado de la ranura ([sistema de actualización HAOS](https://developers.home-assistant.io/docs/operating-system/update-system/)). Este mecanismo protege la ruta de actualización del SO, no automáticamente la configuración de Core, los datos de Recorder ni el estado de red de radio.

Una vez conocidas la ejecución y la ruta de datos, puede determinarse el alcance de la copia de seguridad. Configuración, registros, secretos, base de datos y estados de complementos deben corresponder conjuntamente al modelo de instalación elegido.

## Copia de seguridad y recuperación

Home Assistant puede escribir copias de seguridad automáticas y manuales, cifradas, en destinos locales o externos. La [guía oficial de Backup y Restore](https://www.home-assistant.io/common-tasks/general/) describe ubicaciones de copia de seguridad, Emergency Kit, descarga, restauración durante Onboarding y migración a otro hardware. Desde 2026 se ha modernizado el modelo criptográfico de las copias de seguridad; el [anuncio del cifrado de copias de seguridad](https://www.home-assistant.io/blog/2026/03/26/modernizing-encryption-of-home-assistant-backups/) documenta el cambio de formato y los límites de compatibilidad.

Una copia de seguridad solo es completa en relación con el modelo de instalación:

| Objeto | Copia de seguridad HAOS | Responsabilidad de Container/externa |
|---|---|---|
| Configuración y registros de Core | Incluibles | Respaldar volumen de configuración |
| Apps de Supervisor | Datos de apps incluibles | Contenedores y volúmenes independientes |
| Recorder SQLite | En el área de configuración | Copia de seguridad coherente de BD con BD externa |
| Broker MQTT | Solo con la selección adecuada de app | Configuración y persistencia del broker por separado |
| Zigbee/Z-Wave/Matter | Datos de integración parcialmente | Comprobar por separado copia de controlador/servidor y claves |
| TLS/proxy/DNS | Solo si están dentro de los datos seleccionados | Infraestructura externa por separado |
| Claves de copia de seguridad | No suficientes dentro de la propia copia cifrada | Conservar Emergency Kit por separado |

Una prueba de restauración no termina en el inicio de sesión. Los criterios de aceptación son: Config Entries cargados, registros coherentes, acceso de usuario posible, base de datos sin errores, broker y puentes conectados, dispositivos de radio controlables, automatizaciones críticas probadas y acceso remoto disponible con certificado correcto. Los dispositivos a batería pueden dormir inicialmente tras una migración; un valor inmediato ausente no se considera precipitadamente pérdida de datos.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Backupinventar und Prüfsummen">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ChildItem .\ha-backups -File -Recurse |
  Get-FileHash -Algorithm SHA256 |
  Export-Csv .\ha-backups-manifest.csv -NoTypeInformation
Get-Content .\ha-backups-manifest.csv -First 5</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">find ./ha-backups -type f -print0 | sort -z | xargs -0 sha256sum &gt; ha-backups-manifest.sha256
head -n 5 ha-backups-manifest.sha256
tar -tf ./ha-backups/example-backup.tar | head</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash), [`Export-Csv`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv) y [`Get-Content`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content) crean o leen el manifiesto de Windows. [`find`](https://man7.org/linux/man-pages/man1/find.1.html), [`sort`](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html), [`xargs`](https://man7.org/linux/man-pages/man1/xargs.1.html), [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) y [`tar`](https://www.gnu.org/software/tar/manual/html_node/index.html) se encargan de Unix. Una suma de comprobación prueba que el archivo no ha cambiado; solo la prueba de restauración prueba la posibilidad de descifrado y la recuperación funcional.

## RPO, RTO y alta disponibilidad

En la operación habitual, Home Assistant es una instancia única con estado. Dos instancias activas de Core frente a los mismos dispositivos, registros o comandos de broker no crean alta disponibilidad coordinada automáticamente. Las automatizaciones duplicadas pueden conmutar actuadores varias veces; los controladores de radio y dispositivos locales a menudo solo permiten una propiedad activa.

Un modelo de resiliencia realista combina:

- nodo único fiable o VM con recursos supervisados,
- SAI y almacenamiento adecuado en lugar de medios flash sensibles,
- copias de seguridad separadas, automáticas y cifradas,
- hardware de sustitución documentado o plataforma de destino de VM,
- estados y claves exportables de controladores de radio,
- servicios externos reproducibles,
- restauración controlada con adopción inequívoca de dispositivos y red.

El **RPO** depende de la última copia asegurada de configuración, registro, apps y servicios externos. El historial de Recorder puede tener un RPO distinto al de la configuración de automatizaciones. El **RTO** no comprende solo el arranque de Core, sino también DNS, proxy, base de datos, broker, controlador de radio, reconexión de dispositivos, sensores dormidos y pruebas de aceptación.

## Seguridad y límites de confianza

Home Assistant puede controlar puertas, calefacción, sistemas de alarma y flujos de energía. Por tanto, su ámbito de influencia es físico. El diseño de seguridad separa:

- usuarios y administradores,
- sesiones de navegador, Companion App y API,
- Long-Lived Tokens y Webhooks,
- Core y Custom Integrations,
- apps de Supervisor o contenedores externos,
- segmentos de IoT, gestión y usuarios,
- dispositivos locales y nubes de fabricantes,
- redes de radio y sus claves,
- destinos de copia de seguridad y Emergency Kit.

MFA protege cuentas interactivas, pero no un Long-Lived Token robado. La segmentación de red limita el movimiento lateral, pero no debe bloquear sin control los canales necesarios de Discovery y retorno. Las Custom Integrations tienen proximidad de proceso a Core y se tratan como despliegues de código. Los secretos no aparecen ni en Git ni en archivos de diagnóstico o publicaciones de soporte. Los controles generales se encuentran en [endurecimiento](/kb/haertung), las bases de transporte en [TLS](/kb/tls) y los modelos de contrato de API en [APIs](/kb/apis).

## Historia técnica

Home Assistant comenzó en 2013 como proyecto Python de Paulus Schoutsen. La retrospectiva del décimo aniversario describe la evolución desde una pequeña aplicación local de automatización hasta un gran proyecto de código abierto ([10 años de Home Assistant](https://www.home-assistant.io/blog/2023/09/17/10-years-home-assistant/)). El Core de Python y el modelo de integración siguieron siendo el centro funcional, mientras a su alrededor evolucionaban frontend, clientes móviles, Supervisor, HAOS, hardware de dispositivos y opciones en la nube.

Con Hass.io, posteriormente Home Assistant y Home Assistant OS con Supervisor, surgió una pila de appliance compuesta por sistema operativo, gestión de contenedores, Core y servicios adicionales. La separación se ha aclarado terminológicamente varias veces: los «Add-ons» se llaman ahora **Apps**, mientras que las «integraciones» siguen siendo extensiones Python de Core. Estos términos designan límites de ejecución y seguridad distintos.

En 2024, Home Assistant pasó a la fundación sin ánimo de lucro Open Home Foundation; Nabu Casa siguió siendo socio comercial. El anuncio del proyecto sobre el [ecosistema Open Home](https://www.home-assistant.io/blog/2024/08/08/works-with-home-assistant-becomes-part-ohf/) describe propiedad y gobernanza. En 2025, el proyecto redujo las variantes de instalación compatibles a HAOS y Container. Por tanto, la tendencia histórica no se dirige a un clúster distribuido, sino a un Core central más estable con paquetes de ejecución claramente compatibles y servidores de protocolo independientes.

## Lista de comprobación para administradores

Un Dashboard en verde no basta como evidencia operativa. La lista de comprobación vincula instalación, rutas de dispositivos, automatizaciones, almacenamiento de datos y recuperación en una visión global verificable.

- **Instalación:** Documentar HAOS o Container, arquitectura, host, almacenamiento, red y propiedad.
- **Pila:** Separar Core, Supervisor, apps/contenedores externos, base de datos, broker, proxy y servidor de radio.
- **Inventario:** Registrar Config Entry, Device ID, Entity ID, Unique ID, Area y `via_device`.
- **Procedencia de datos:** Marcar Local/Cloud y Push/Polling para cada integración crítica.
- **Estado:** Distinguir `unknown`, `unavailable`, valor obsoleto y éxito confirmado en el dispositivo.
- **Automatización:** Probar Trigger, Context, Condition, Run Mode, comportamiento tras reinicio y compensación.
- **API:** Controlar contexto de usuario, almacenamiento de tokens, WebSocket, proxy inverso y TLS.
- **Recorder:** Supervisar base de datos, filtros, intervalo de commit, Purge, E/S, crecimiento y copia de seguridad.
- **Red IoT:** Comprobar explícitamente mDNS, SSDP, MQTT, VLAN, IPv6 y rutas de radio/puente.
- **Actualizaciones:** Integrar notas de versión, Custom Integrations, copia de seguridad, despliegue y aceptación.
- **Recuperación:** Probar conjuntamente Core, servicios externos, estado de radio, claves y Emergency Kit.
- **Evidencia:** Verificar de extremo a extremo no solo UI y contenedor, sino al menos una ruta de sensor, actuador, automatización y API.

## Fuentes

- [Home Assistant Developer Docs – Visión general de arquitectura](https://developers.home-assistant.io/docs/architecture_index/)
- [Home Assistant Developer Docs – Arquitectura de integraciones](https://developers.home-assistant.io/docs/architecture_components/)
- [Home Assistant Developer Docs – Core architecture](https://developers.home-assistant.io/docs/architecture/core/)
- [Home Assistant – HAOS o Container](https://www.home-assistant.io/faq/ha-vs-hassio/)
- [Home Assistant Developer Docs – Operating System](https://developers.home-assistant.io/docs/operating-system/)
- [Home Assistant Developer Docs – Supervisor](https://developers.home-assistant.io/docs/supervisor/)
- [Home Assistant – Discontinuación de Core, Supervised y 32 bits](https://www.home-assistant.io/blog/2025/05/22/deprecating-core-and-supervised-installation-methods-and-32-bit-systems/)
- [GitHub – Home Assistant Core](https://github.com/home-assistant/core)
- [Home Assistant Developer Docs – Async dependency](https://developers.home-assistant.io/docs/core/integration-quality-scale/rules/async-dependency/)
- [Home Assistant Developer Docs – Integration Quality Scale](https://developers.home-assistant.io/docs/core/integration-quality-scale/)
- [Home Assistant Developer Docs – Config entries](https://developers.home-assistant.io/docs/config_entries_index/)
- [Home Assistant Developer Docs – Device registry](https://developers.home-assistant.io/docs/device_registry_index/)
- [Home Assistant Developer Docs – Entity registry](https://developers.home-assistant.io/docs/entity_registry_index/)
- [Home Assistant Developer Docs – Dispositivos y servicios](https://developers.home-assistant.io/docs/architecture/devices-and-services/)
- [Home Assistant – Integraciones](https://www.home-assistant.io/integrations/)
- [Home Assistant – Zeroconf](https://www.home-assistant.io/integrations/zeroconf/)
- [Home Assistant – SSDP](https://www.home-assistant.io/integrations/ssdp/)
- [Home Assistant – MQTT](https://www.home-assistant.io/integrations/mqtt/)
- [Home Assistant – ZHA](https://www.home-assistant.io/integrations/zha/)
- [Home Assistant – Z-Wave JS](https://www.home-assistant.io/integrations/zwave_js/)
- [Home Assistant – Matter](https://www.home-assistant.io/integrations/matter/)
- [Home Assistant – Thread](https://www.home-assistant.io/integrations/thread/)
- [Home Assistant – Fundamentos de automatizaciones](https://www.home-assistant.io/docs/automation/basics/)
- [Home Assistant – Automation actions](https://www.home-assistant.io/docs/automation/action/)
- [Home Assistant – Automation triggers](https://www.home-assistant.io/docs/automation/trigger/)
- [Home Assistant – Conditions](https://www.home-assistant.io/docs/scripts/conditions/)
- [Home Assistant – Automation templating](https://www.home-assistant.io/docs/automation/templating/)
- [Home Assistant – Configuración](https://www.home-assistant.io/docs/configuration/)
- [Home Assistant – Packages](https://www.home-assistant.io/docs/configuration/packages/)
- [Home Assistant – Asegurar Home Assistant](https://www.home-assistant.io/docs/configuration/securing/)
- [Home Assistant – Recorder](https://www.home-assistant.io/integrations/recorder/)
- [Home Assistant Developer Docs – REST API](https://developers.home-assistant.io/docs/api/rest/)
- [Home Assistant Developer Docs – WebSocket API](https://developers.home-assistant.io/docs/api/websocket/)
- [Home Assistant Developer Docs – Authentication API](https://developers.home-assistant.io/docs/auth_api/)
- [Microsoft – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [Microsoft – ConvertTo-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertto-json)
- [curl – Manual](https://curl.se/docs/manpage.html)
- [jq – Manual](https://jqlang.org/manual/)
- [Home Assistant – HTTP integration](https://www.home-assistant.io/integrations/http/)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [Microsoft – Get-NetTCPConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Eclipse Mosquitto – mosquitto_sub](https://mosquitto.org/man/mosquitto_sub-1.html)
- [Eclipse Mosquitto – mosquitto_pub](https://mosquitto.org/man/mosquitto_pub-1.html)
- [Docker – inspect](https://docs.docker.com/reference/cli/docker/inspect/)
- [Docker – logs](https://docs.docker.com/reference/cli/docker/container/logs/)
- [Docker – stats](https://docs.docker.com/reference/cli/docker/container/stats/)
- [Microsoft – ConvertFrom-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json)
- [Microsoft – Select-String](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/select-string)
- [GNU Grep – Manual](https://www.gnu.org/software/grep/manual/grep.html)
- [GNU Coreutils – df](https://www.gnu.org/software/coreutils/manual/html_node/df-invocation.html)
- [GNU Coreutils – du](https://www.gnu.org/software/coreutils/manual/html_node/du-invocation.html)
- [Home Assistant – System Health](https://www.home-assistant.io/integrations/system_health/)
- [Home Assistant – Logger](https://www.home-assistant.io/integrations/logger/)
- [Home Assistant Developer Docs – HAOS update system](https://developers.home-assistant.io/docs/operating-system/update-system/)
- [Home Assistant – Backup y Restore](https://www.home-assistant.io/common-tasks/general/)
- [Home Assistant – Cifrado de copias de seguridad modernizado](https://www.home-assistant.io/blog/2026/03/26/modernizing-encryption-of-home-assistant-backups/)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [Microsoft – Export-Csv](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv)
- [Microsoft – Get-Content](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-content)
- [Linux man-pages – find(1)](https://man7.org/linux/man-pages/man1/find.1.html)
- [GNU Coreutils – sort](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html)
- [Linux man-pages – xargs(1)](https://man7.org/linux/man-pages/man1/xargs.1.html)
- [GNU Coreutils – utilidades sha2](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [GNU Tar – Manual](https://www.gnu.org/software/tar/manual/html_node/index.html)
- [Home Assistant – 10 años de Home Assistant](https://www.home-assistant.io/blog/2023/09/17/10-years-home-assistant/)
- [Home Assistant – Open Home Foundation y gobernanza](https://www.home-assistant.io/blog/2024/08/08/works-with-home-assistant-becomes-part-ohf/)
