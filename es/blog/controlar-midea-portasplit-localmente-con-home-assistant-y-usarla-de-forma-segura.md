---
title: "Controlar Midea PortaSplit localmente con Home Assistant y usarla de forma segura"
navTitle: "Configurar PortaSplit"
description: "Desde la integración comunitaria adecuada hasta la VLAN de IoT: así se configura la PortaSplit, se protegen el token y la clave, y se limitan los accesos a la nube y a la red."
date: "2026-07-24"
kategorie: "Hogar inteligente e IoT"
timeToRead: "14 min de lectura"
themen:
  - "smart-home-iot"
related:
  - "midea-portasplit-home-assistant"
  - "serverloser-newsletter-cloudflare-workers-d1"
slug: "controlar-midea-portasplit-localmente-con-home-assistant-y-usarla-de-forma-segura"
translationOf: "midea-portasplit-home-assistant-einrichten"
url: "https://rafaelpfister.ch/es/blog/controlar-midea-portasplit-localmente-con-home-assistant-y-usarla-de-forma-segura"
---

La Midea PortaSplit puede controlarse directamente en la red local mediante Home Assistant después de configurarla. Para ello, la integración comunitaria necesita dos credenciales específicas del dispositivo procedentes de la nube de Midea: token y clave.

Este artículo explica la selección, configuración y protección de la integración. Las soluciones descritas proceden de la comunidad y no cuentan con soporte oficial ni de Midea ni de Home Assistant. Por ello, los cambios de firmware o de la nube pueden afectar a su funcionamiento en cualquier momento. El contexto sobre la interfaz de tokens y la advertencia ambigua sobre su desactivación se encuentra en el [análisis de las API de la nube de Midea](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Cómo funciona el control local

Una vez configurados, los comandos de control se envían directamente de Home Assistant a la PortaSplit:

```text
Home Assistant → lokales Netzwerk → Midea PortaSplit
```

Un comando de conmutación no tiene que pasar por un servidor externo de Midea, el tiempo de respuesta es breve, una avería de la nube de Midea no paraliza necesariamente el control local ya configurado y, en principio, el dispositivo sigue siendo controlable incluso sin acceso a Internet.

Sin embargo, en los dispositivos más recientes con el denominado protocolo V3, la PortaSplit no acepta comandos locales sin protección. Home Assistant necesita dos valores específicos del dispositivo, un token y una clave, que sirven para autenticar y cifrar la conexión local. Durante la configuración inicial, la integración los obtiene una única vez a través de una interfaz de la nube de Midea y luego los guarda localmente; no se requiere una conexión a la nube para el control posterior.

Simplificado, el proceso es el siguiente:

1. La PortaSplit se conecta a MSmartHome.
2. Home Assistant inicia sesión en una nube de Midea.
3. Home Assistant obtiene el ID del dispositivo, el token y la clave.
4. El token y la clave se guardan localmente.
5. Home Assistant controla la PortaSplit directamente en la LAN.

## Qué integración es adecuada

### Midea Smart AC

El repositorio <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a> se centra en los aparatos de aire acondicionado Midea y modelos OEM relacionados, y admite los tipos de dispositivo `0xAC` y `0xCC`. Ofrece control local, configuración gráfica, detección automática, configuración manual con token y clave, así como consulta automática de las capacidades del dispositivo. El «Out Silent Mode» de la PortaSplit cuenta con soporte explícito.

Como indicio de compatibilidad, el proyecto menciona, entre otras, las aplicaciones Artic King, Midea Air, NetHome Plus, SmartHome o MSmartHome, Toshiba AC NA y 美的美居. En Europa, la PortaSplit utiliza habitualmente MSmartHome y, por tanto, encaja en este ecosistema.

### Midea AC LAN

El repositorio <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a> no solo admite equipos de aire acondicionado, sino también muchas otras categorías de dispositivos Midea: deshumidificadores, ventiladores, purificadores de aire, lavadoras, secadoras, lavavajillas, equipos de agua caliente, bombas de calor, frigoríficos y más, en parte también bajo marcas de terceros como Carrier o Electrolux. También ofrece comunicación local, detección automática de dispositivos y sensores adicionales y, según la descripción del proyecto, mantiene una conexión TCP más prolongada con el dispositivo para sincronizar rápidamente los cambios de estado. Requiere al menos Home Assistant 2024.4.1.

La mayor desventaja actualmente es la advertencia del desarrollador: las API de tokens en la nube utilizadas para añadir nuevos dispositivos se están desactivando gradualmente. Esto podría imposibilitar añadir nuevos dispositivos más adelante.

### Recomendación

Para una instalación exclusivamente con PortaSplit, empezaría con `Midea Smart AC` y tendría `Midea AC LAN` presente como alternativa. `Midea Smart AC` está más orientada a equipos de aire acondicionado y documenta explícitamente las funciones actuales de la PortaSplit.

No tiene sentido utilizar ambas integraciones a la vez y de forma permanente con el mismo dispositivo. Varias conexiones simultáneas provocan problemas de estado, tráfico de red innecesario y un comportamiento difícil de seguir.

## Qué aporta la integración

Después de la configuración, la PortaSplit aparece como una entidad de tipo `climate` en Home Assistant. Según el firmware y la integración, están disponibles, entre otras, las siguientes funciones:

- Encendido y apagado
- Ajustar la temperatura objetivo
- Leer la temperatura ambiente actual
- Refrigeración, deshumidificación y funcionamiento exclusivo del ventilador
- Ajustar la velocidad del ventilador
- Controlar la función Swing
- Modo Eco y Boost
- Leer la humedad del aire
- Mostrar códigos de error
- Leer valores de energía y potencia
- Mostrar valores del compresor
- Activar el modo silencioso de la unidad exterior

Las entidades que aparecen realmente dependen del modelo, el firmware, el protocolo utilizado y la integración correspondiente. `Midea Smart AC` consulta las capacidades comunicadas por el dispositivo y oculta las funciones que el modelo no admite. `Midea AC LAN` también documenta amplias entidades de climatización, entre ellas temperatura, humedad del aire, potencia actual, energía total, frecuencia del compresor, estado de la bomba y diversos modos de funcionamiento, y menciona métodos propios para decodificar los datos de energía de determinados subtipos de PortaSplit.

No todas las mediciones mostradas tienen por qué ser correctas. En particular, el consumo energético y la potencia se transmiten en formatos distintos en los diferentes modelos de Midea. Si Home Assistant muestra valores claramente erróneos, normalmente debe ajustarse el método de decodificación utilizado y el dispositivo no está averiado.

## Requisitos

Se necesita una Midea PortaSplit con función Wi-Fi, una red Wi-Fi de 2,4 GHz, la aplicación MSmartHome, una cuenta de usuario de Midea, Home Assistant, HACS y acceso de red entre Home Assistant y la PortaSplit. Primero debe conectarse la PortaSplit normalmente con la aplicación MSmartHome y solo después añadirla a Home Assistant.

## Paso 1: conectar la PortaSplit a MSmartHome

1. Instalar la aplicación MSmartHome.
2. Crear una cuenta de Midea o iniciar sesión.
3. Poner la PortaSplit en modo de emparejamiento Wi-Fi.
4. Conectar el dispositivo a la red Wi-Fi de 2,4 GHz.
5. Comprobar que la PortaSplit se puede controlar mediante la aplicación.

Muchos dispositivos IoT siguen admitiendo únicamente 2,4 GHz. Si el router utiliza el mismo SSID para 2,4 y 5 GHz, la configuración suele funcionar de todos modos. Si hay problemas, puede ayudar proporcionar temporalmente una red Wi-Fi independiente de 2,4 GHz.

## Paso 2: instalar HACS

HACS es la tienda comunitaria de Home Assistant. Permite instalar integraciones comunitarias que no forman parte de Home Assistant Core. Tras instalar HACS, se abre HACS, se accede a las integraciones, se busca `Midea Smart AC`, se descarga la integración y se reinicia Home Assistant. Como alternativa, se puede buscar `Midea AC LAN`.

HACS simplifica la instalación y las actualizaciones. Sin embargo, no convierte una integración personalizada en un componente de Home Assistant revisado oficialmente. Esta diferencia es importante desde el punto de vista de la seguridad y se aborda más abajo.

## Paso 3: añadir Midea Smart AC

Después de reiniciar, vaya a Ajustes, Dispositivos y servicios y Añadir integración, busque `Midea Smart AC` y, a continuación, seleccione `Discover devices`. La integración puede buscar en toda la red local o consultar específicamente la dirección IP de la PortaSplit.

Si se encuentra el dispositivo, en los dispositivos V3 más recientes la integración necesita la región, la cuenta de Midea, la contraseña y el ID del dispositivo, así como el token y la clave derivados de ellos. La región de la nube debe coincidir con la cuenta utilizada. En caso de problemas, el proyecto recomienda probar también las demás regiones disponibles.

### Configuración manual

Si la configuración automática falla, el dispositivo puede configurarse manualmente. Para `Midea Smart AC` se requieren los siguientes datos:

```text
Device ID
IP-Adresse
Port
Gerätetyp
Token
Key
```

El puerto predeterminado documentado es:

```text
6444/TCP
```

Para los dispositivos V3, la documentación indica que el token es una cadena hexadecimal de 128 caracteres y la clave una cadena hexadecimal de 64 caracteres. Ambos valores son secretos y deben tratarse en consecuencia. Quien no quiera obtener las credenciales mediante Discovery puede recuperarlas con su propia cuenta usando la CLI `msmart-ng`.

## Usar la PortaSplit de forma segura

Quien controla la PortaSplit localmente recupera parte del control de la nube del fabricante, pero también traslada la responsabilidad a su propia red. Los siguientes puntos ayudan a que el token y la clave causen poco daño incluso ante un incidente y a que el dispositivo permanezca correctamente aislado.

### El token y la clave son secretos

El token y la clave autentican la comunicación local con el dispositivo y deben tratarse como una contraseña. Para el funcionamiento, lo más importante es que no figuren en registros, copias de seguridad sin cifrar ni repositorios.

### Sin redirección de puertos hacia la PortaSplit

El error evitable más frecuente sería hacer que el puerto local del dispositivo fuera accesible directamente desde Internet. Una regla como esta sería peligrosa:

```text
Internet → TCP 6444 → PortaSplit
```

No hay una buena razón para hacer accesible la PortaSplit directamente desde Internet. Home Assistant ya se encuentra en la red local y actúa como instancia de control. El router no debe tener ninguna redirección de puertos hacia la PortaSplit, UPnP debe restringirse o desactivarse si es posible, las conexiones entrantes deben bloquearse de forma predeterminada y no debe utilizarse una autorización DMZ para el dispositivo.

### VLAN de IoT propia

La mejor arquitectura de red es una red de IoT independiente:

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

La PortaSplit se encuentra en la VLAN de IoT. Home Assistant puede acceder específicamente al dispositivo, pero la PortaSplit no puede acceder libremente a PC, NAS y otros sistemas internos. Una posible lógica de firewall:

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Durante la configuración inicial, el dispositivo necesita acceso a Internet a la nube de Midea. Tras configurar correctamente el control local, se puede probar si es posible bloquear el acceso saliente a Internet. No debe aplicarse inmediatamente un bloqueo definitivo. Primero hay que comprobar si el control local sigue funcionando, si el dispositivo permanece accesible tras reiniciarlo, si resiste un reinicio del router, si sigue respondiendo incluso después de varios días, si la aplicación MSmartHome sigue siendo necesaria y si siguen ofreciéndose actualizaciones de firmware. Quien quiera seguir utilizando la nube y las actualizaciones de firmware puede permitir temporalmente el acceso saliente a Internet y bloquearlo de nuevo después.

### La segmentación de red puede impedir Discovery

La búsqueda automática de dispositivos se basa a menudo en tráfico broadcast o multicast, que normalmente no se enruta a través de los límites de las VLAN. Por ello, es posible que Home Assistant no encuentre automáticamente la PortaSplit aunque se permita una conexión IP normal.

En ese caso, ayuda configurar temporalmente la PortaSplit en la misma VLAN que Home Assistant, indicar manualmente la IP del dispositivo, utilizar una función adecuada de retransmisión de broadcast o definir reglas de firewall específicas después de la configuración. Desde el punto de vista de la seguridad, la configuración manual suele ser incluso la mejor opción, ya que no es necesario permitir tráfico broadcast adicional entre las redes.

### Asignación DHCP estática

La PortaSplit debe recibir una asignación DHCP fija en el router:

```text
PortaSplit → 192.168.30.25
```

Una reserva DHCP suele ser preferible a una IP estática configurada en el dispositivo. Home Assistant encuentra el dispositivo de forma fiable, las reglas de firewall pueden limitarse a una dirección fija, el análisis de errores se simplifica y la asignación se mantiene estable después de reiniciar el router o el dispositivo. Así, una regla de firewall puede formularse de forma muy restrictiva:

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

El puerto realmente necesario debe verificarse con la integración y el propio dispositivo.

### Home Assistant como ancla central de confianza

Quien controla la PortaSplit localmente traslada parte de la confianza desde la nube de Midea a Home Assistant. Si Home Assistant se ve comprometido, un atacante podría controlar no solo el aire acondicionado, sino todo el hogar inteligente.

Por ello, Home Assistant debe actualizarse periódicamente, no debe publicarse mediante una redirección de puertos sin protección, debe estar protegido con una contraseña fuerte y única, utilizar autenticación multifactor, crear copias de seguridad cifradas, contener solo los complementos necesarios y no permitir acceso SSH innecesario desde Internet. Para el acceso remoto, una VPN, Home Assistant Cloud o un proxy inverso correctamente configurado son mejores opciones que una simple redirección de puertos al puerto 8123.

### HACS y el riesgo de cadena de suministro

`Midea Smart AC` y `Midea AC LAN` son integraciones personalizadas. Se ejecutan dentro de Home Assistant y, por tanto, tienen amplio acceso a su entorno de ejecución. Una integración maliciosa o comprometida podría teóricamente leer datos de configuración, extraer secretos, establecer conexiones de red, escanear dispositivos en la red local, leer estados de otras entidades, transferir datos a sistemas externos y afectar a la disponibilidad de Home Assistant.

Esto no significa que las integraciones mencionadas sean maliciosas. Ambos proyectos son públicos, se desarrollan activamente y cuentan con una comunidad visible. Sin embargo, el código abierto no es una garantía automática de seguridad. Antes de instalar, conviene al menos comprobar si el repositorio se mantiene activamente, si hay lanzamientos regulares, cuántas personas contribuyen al código, si existen problemas de seguridad abiertos, si recientemente han cambiado los mantenedores o propietarios del repositorio, si HACS apunta al repositorio esperado y si una actualización incluye cambios inusualmente grandes o inexplicables.

Las actualizaciones no deben instalarse a ciegas inmediatamente después de su publicación. Especialmente en sistemas de hogar inteligente críticos para la seguridad, conviene esperar unos días y revisar las notas de la versión y los problemas reportados.

### Proteger la cuenta en la nube

Mientras se utilice la nube de Midea para la configuración o el control mediante la aplicación, la cuenta de Midea también sigue formando parte del modelo de seguridad. Debe tener una contraseña única que no se comparta con otros servicios, un gestor de contraseñas, autenticación multifactor si se ofrece, eliminación de teléfonos inteligentes y sesiones antiguas, evitar cuentas compartidas y revisar periódicamente qué dispositivos están registrados en la cuenta.

Si la integración de Home Assistant solicita nombre de usuario y contraseña durante la configuración, debe comprobarse si las credenciales se utilizan solo para obtener el token una vez o si se guardan permanentemente. Los desarrolladores de `Midea Smart AC` escriben que los dispositivos no se vinculan a cuentas integradas de la integración después de configurarlos y que el token y la clave también pueden obtenerse manualmente mediante la CLI con la propia cuenta. Siempre que sea posible, debe preferirse la cuenta propia a cuentas colectivas externas o integradas.

### ¿Bloquear la nube o no?

Después de una configuración correcta surge la pregunta de si debe bloquearse completamente el acceso a Internet de la PortaSplit. A favor del bloqueo están una menor telemetría, menor dependencia de servicios externos, una vía de ataque más reducida a través de la nube del fabricante, el hecho de que el dispositivo no pueda contactar con destinos externos arbitrarios y un menor efecto de los cambios en la nube.

En contra está que la aplicación MSmartHome podría dejar de funcionar fuera de la red doméstica, que no se descargarían actualizaciones de firmware, que podrían fallar las funciones de hora o de nube, que un nuevo inicio de sesión o una restauración serían más difíciles y que algunos dispositivos reaccionan de forma inesperada tras largos periodos sin conexión.

Una secuencia pragmática: configurar el dispositivo normalmente, probar Home Assistant y la aplicación, guardar el token y la configuración, bloquear el acceso a Internet, reiniciar el dispositivo y Home Assistant, observar durante varios días y, si es necesario, volver a permitir el acceso a Internet solo temporalmente.

### Actualizaciones de firmware: ¿mejora de seguridad o riesgo de integración?

Las actualizaciones de firmware son un dilema en los dispositivos IoT. Pueden corregir vulnerabilidades conocidas, mejorar la estabilidad, modernizar mecanismos de seguridad y aportar nuevas funciones. Pero también pueden cambiar interfaces locales, romper integraciones basadas en ingeniería inversa, invalidar tokens, desactivar la API local e introducir nuevas dependencias de la nube.

Por ejemplo, el firmware de PortaSplit distribuido en enero de 2026 incorporó un nuevo modo silencioso para la unidad exterior que reduce el ruido en unos 6 decibelios. Las integraciones comunitarias tuvieron primero que analizarlo e implementarlo, como se documenta en un issue de GitHub específico para la PortaSplit.

De ello se deduce: no impedir las actualizaciones de firmware por principio; antes de actualizar, comprobar si otros usuarios de Home Assistant informan de problemas, guardar previamente la configuración y el token, crear una copia de seguridad de Home Assistant y probar por completo el control local después de la actualización. Seguridad no significa «no actualizar nunca». Un firmware obsoleto puede ser más peligroso que una integración temporalmente incompatible.

### Los registros de depuración contienen datos sensibles

En caso de problemas, los proyectos de código abierto suelen solicitar registros de depuración. La documentación de `Midea AC LAN` muestra cómo activar el registro para los dos componentes relevantes:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Después, los registros se pueden descargar mediante Ajustes, Sistema y Registros. Según la integración y el caso de error, estos registros pueden contener direcciones IP locales, ID del dispositivo, número de serie, identificador de modelo, respuestas de la nube, información de la cuenta, token o partes de él, paquetes de red, así como marcas de tiempo y patrones de uso. Por ello, deben revisarse y censurarse los valores sensibles antes de subirlos a un issue público de GitHub.

Una vez finalizada la resolución del problema, debe eliminarse de nuevo el registro de depuración. Mantenerlo activado permanentemente no solo aumenta el consumo de almacenamiento, sino también la cantidad de información sensible en las copias de seguridad.

### Lo que dice Midea sobre la seguridad

Midea promociona su ecosistema SmartHome con orientación a varios estándares de seguridad y privacidad, entre ellos EN 303 645, UK PSTI, NIST, tratamiento de datos conforme al RGPD y requisitos de la Directiva de equipos radioeléctricos de la UE. Son señales positivas, pero no indican cómo se implementan realmente cada firmware de PortaSplit, cada endpoint de nube y cada API local. Las afirmaciones de certificación y marketing no sustituyen una revisión técnica del dispositivo concreto.

Del mismo modo, sería erróneo deducir de la advertencia de una integración comunitaria que la PortaSplit sea insegura en general. El problema descrito afecta a la arquitectura de los tokens persistentes y a su uso por clientes no oficiales.

### Riesgo según el escenario

| Escenario | Riesgo | Motivo |
| --- | --- | --- |
| Red doméstica normal sin redirección de puertos | manejable | Un atacante necesita primero acceso a la Wi-Fi, Home Assistant o una copia de seguridad. |
| Red doméstica plana con muchos dispositivos IoT inseguros | medio | Otro dispositivo IoT comprometido puede acceder a PortaSplit o Home Assistant en la misma red. |
| PortaSplit accesible directamente desde Internet | alto | El dispositivo nunca debe publicarse mediante redirección de puertos. |
| Token y clave públicos en GitHub | alto | Los secretos deben considerarse comprometidos; no está garantizado que puedan revocarse. |
| VLAN de IoT independiente, firewall restrictivo, control local | comparativamente bajo | Incluso si existe una vulnerabilidad en el dispositivo, la libertad de movimiento en la red queda muy limitada. |

## Copia de seguridad de la configuración

Guardar el token, la clave y la configuración es el paso único más importante: una vez cerradas las interfaces de tokens en la nube, una copia de seguridad será la única forma de realizar una nueva configuración. `Midea AC LAN` guarda un archivo de configuración JSON para dispositivos V3 después de una configuración correcta. La ruta documentada es:

```text
/config/.storage/midea_ac_lan/
```

El archivo lleva como nombre el ID del dispositivo:

```text
<device-id>.json
```

Este archivo no es una nota de texto normal. Puede contener el ID del dispositivo, número de serie, dirección IP, token, clave, información del protocolo y parámetros de la nube y del dispositivo. Por tanto:

- No subirlo a un repositorio público de GitHub.
- No publicarlo en foros.
- No compartirlo como captura de pantalla sin censurar.
- No enviarlo por correo electrónico sin cifrar.

Incluso un repositorio Git privado no es automáticamente el lugar adecuado para almacenarlo, porque los secretos permanecen en el historial de Git aunque después se eliminen del archivo actual. Son más adecuados una copia de seguridad cifrada, un gestor de contraseñas con archivo adjunto, una copia de seguridad cifrada en NAS, un medio sin conexión cifrado o un archivo cifrado con una contraseña guardada por separado.

Para hacer una copia de seguridad mediante el terminal de Home Assistant:

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Mostrar archivo:

```bash
cat <device-id>.json
```

Para copiarlo, el archivo no debe transferirse mediante un servicio web público. Es mejor crear un archivo cifrado que luego se traslade a una copia de seguridad cifrada:

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

Los archivos de `.storage` no deben editarse manualmente. El desarrollador recomienda expresamente no eliminar ni modificar directamente el archivo JSON en caso de problemas, sino renombrarlo y guardarlo antes de realizar cambios.

Una copia de seguridad completa de Home Assistant también contiene estos archivos. Aun así, tiene sentido disponer de una copia independiente, porque las copias de seguridad de Home Assistant pueden dañarse, una restauración puede sobrescribir la integración, el archivo podría ser necesario específicamente para una nueva configuración posterior y una copia de seguridad nunca debe estar solo en el mismo sistema.

## Eliminar secretos de un repositorio Git publicado

Si se publicó por error un archivo JSON en GitHub, no basta con eliminarlo normalmente y crear un nuevo commit. El archivo sigue siendo recuperable en el historial de Git. Como mínimo, son necesarios estos pasos:

1. Poner inmediatamente el repositorio como privado, si es posible.
2. Eliminar el archivo de todo el historial de Git.
3. Tener en cuenta las cachés y forks de GitHub.
4. Tratar el token como comprometido.
5. Eliminar el dispositivo de la cuenta de Midea y volver a conectarlo si eso genera nuevas claves.
6. Configurar de nuevo la integración de Home Assistant.
7. Cambiar la contraseña de la cuenta de Midea si las credenciales también se vieron afectadas.

Que el emparejamiento de nuevo genere realmente un nuevo token varía según el dispositivo y la arquitectura de la nube. No debe confiarse en que cambiar la contraseña de la cuenta invalide automáticamente el token local del dispositivo.

## Automatizaciones útiles

Tras una integración correcta, la PortaSplit puede funcionar de forma mucho más inteligente. Los ID de entidad deben adaptarse a la propia instalación.

Refrigerar solo con las ventanas cerradas:

```yaml
alias: PortaSplit nur bei geschlossenen Fenstern
triggers:
  - trigger: state
    entity_id: binary_sensor.wohnzimmer_fenster
    to: "on"

actions:
  - action: climate.turn_off
    target:
      entity_id: climate.portasplit
```

Encender cuando la temperatura ambiente sea alta:

```yaml
alias: PortaSplit bei Hitze einschalten
triggers:
  - trigger: numeric_state
    entity_id: sensor.wohnzimmer_temperatur
    above: 27

conditions:
  - condition: state
    entity_id: binary_sensor.wohnzimmer_fenster
    state: "off"
  - condition: state
    entity_id: person.rafael
    state: "home"

actions:
  - action: climate.set_hvac_mode
    target:
      entity_id: climate.portasplit
    data:
      hvac_mode: cool

  - action: climate.set_temperature
    target:
      entity_id: climate.portasplit
    data:
      temperature: 24
```

Preenfriar antes de ir a dormir:

```yaml
alias: Schlafzimmer vorkühlen
triggers:
  - trigger: time
    at: "21:00:00"

conditions:
  - condition: numeric_state
    entity_id: sensor.schlafzimmer_temperatur
    above: 25

actions:
  - action: climate.set_temperature
    target:
      entity_id: climate.portasplit
    data:
      temperature: 23
```

Apagar cuando no haya nadie en casa:

```yaml
alias: PortaSplit bei Abwesenheit ausschalten
triggers:
  - trigger: state
    entity_id: zone.home
    to: "0"
    for:
      minutes: 10

actions:
  - action: climate.turn_off
    target:
      entity_id: climate.portasplit
```

## Configuración recomendada de un vistazo

```text
1. PortaSplit mit MSmartHome einrichten
2. Midea Smart AC über HACS installieren
3. PortaSplit automatisch oder manuell hinzufügen
4. DHCP-Reservation erstellen
5. Home-Assistant-Backup anfertigen
6. Token- und Konfigurationsdaten verschlüsselt sichern
7. PortaSplit in ein separates IoT-VLAN verschieben
8. Zugriff von Home Assistant zur PortaSplit erlauben
9. Zugriff der PortaSplit auf interne Netze blockieren
10. Internetzugriff testweise blockieren
11. lokale Steuerung nach Neustarts prüfen
12. Firmware- und Integrationsupdates kontrolliert durchführen
```

La dirección de comunicación deseada es la siguiente:

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

## Estado operativo recomendado

La Midea PortaSplit se integra sorprendentemente bien en Home Assistant. Tras una configuración correcta, puede controlarse localmente e incorporarse en automatizaciones, lo que elimina gran parte de la dependencia de la nube para el funcionamiento diario.

Desde el punto de vista de la seguridad, la integración es aceptable si se siguen algunas reglas básicas: no redirigir puertos, mantener secretos el token y la clave, cifrar las copias de seguridad, revisar los registros de depuración antes de publicarlos, proteger Home Assistant, segmentar los dispositivos IoT, limitar el acceso saliente a Internet a lo necesario y no instalar actualizaciones de firmware y HACS a ciegas. Utilizada de este modo, la PortaSplit sigue siendo un potente equipo de aire acondicionado y, al mismo tiempo, una parte sensata de un hogar inteligente controlado localmente.

## Fuentes

1.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: integración `Midea Smart AC`: tipos de dispositivo admitidos `0xAC` y `0xCC`, PortaSplit con «Out Silent Mode», uso de la nube para obtener el token y la clave en dispositivos V3, configuración manual y puerto predeterminado 6444.

2.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: integración `Midea AC LAN`: categorías de dispositivos admitidas, conexión TCP más prolongada para sincronizar el estado y versión mínima Home Assistant 2024.4.1.

3.  [midea_ac_lan: documentación de las entidades de climatización](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/AC.md): entidades y atributos para equipos de aire acondicionado, incluidos potencia, energía total, frecuencia del compresor y métodos de decodificación para los valores energéticos de subtipos individuales.

4.  [midea_ac_lan: indicaciones de depuración y configuración](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): ubicación de la configuración del dispositivo en `/config/.storage/midea_ac_lan/`, recomendación de guardar en lugar de eliminar el archivo JSON y configuración de logger para registros de depuración.

5.  [Issue 779: Out Silent Mode de la PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/779): solicitud de soporte para el modo silencioso de la unidad exterior introducido con la actualización de firmware de enero de 2026, que reduce el ruido en unos 6 decibelios.

6.  [Midea SmartHome](https://www.midea.com/global/smarthome): información del fabricante sobre los estándares de seguridad y privacidad EN 303 645, PSTI, NIST, RGPD y RED DA.

7.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): instalación y gestión de integraciones personalizadas que no forman parte de Home Assistant Core.
