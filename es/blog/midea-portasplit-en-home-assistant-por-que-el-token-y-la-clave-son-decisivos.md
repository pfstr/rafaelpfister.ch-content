---
title: "Proteger Midea PortaSplit en Home Assistant: token, clave y red doméstica"
navTitle: "Proteger PortaSplit"
description: "El token y la clave de PortaSplit proceden de la nube de Midea y no caducan. Así puede proteger estos valores, aislar el dispositivo en la red doméstica y mantener actualizados de forma controlada Home Assistant, la integración y el firmware."
date: "2026-07-24"
kategorie: "Home Assistant e IoT"
timeToRead: "14 min de lectura"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant
  - midea-v2-cloud-api-portasplit-home-assistant
image: "../images/midea-portasplit-home-assistant/portasplit-dashboard.png"
slug: "midea-portasplit-en-home-assistant-por-que-el-token-y-la-clave-son-decisivos"
translationOf: "midea-portasplit-home-assistant-absichern"
translationId: article-a02e26cce22063f1
translationReview: automatic
translationSourceHash: d9e59ab5647df261814123d662df93c68c9f6cd35c20cec518de0a9cf54d6e22
translatedAt: 2026-10-31T03:38:59.037Z
translationModel: gpt-5.6-terra
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/midea-portasplit-en-home-assistant-por-que-el-token-y-la-clave-son-decisivos
---

<aside class="article-update">
  <p class="article-update__label">Lo que los propietarios de PortaSplit deberían hacer ahora</p>
  <p>Home Assistant obtiene el token y la clave de PortaSplit mediante interfaces privadas en la nube durante la configuración. El proyecto Midea AC LAN advierte desde el 19 de mayo de 2025 de posibles cambios; no se ha documentado una fecha de desactivación por parte del fabricante. Para los propietarios, esto significa:</p>
  <ol>
    <li><strong>Guardar de forma cifrada el token, la clave y la configuración.</strong> Si más adelante la obtención deja de funcionar, la copia de seguridad será la única forma de restaurarlos.</li>
    <li><strong>No desvincular el dispositivo sin necesidad.</strong> Restablecer los valores de fábrica, eliminarlo de la cuenta de Midea o cambiar el módulo Wi‑Fi obliga a obtener un nuevo token.</li>
    <li><strong>Aislar la PortaSplit en la red doméstica.</strong> Sin redirección de puertos, con una VLAN de IoT propia y acceso solo desde Home Assistant.</li>
  </ol>
</aside>

El control local de la Midea PortaSplit se basa en dos valores específicos del dispositivo: el token y la clave. Autentican la conexión entre Home Assistant y el dispositivo y actualmente solo pueden obtenerse a través de la nube de Midea. Esto implica dos tareas: guardar los valores de forma que una nueva configuración siga siendo posible sin la nube, y operar el dispositivo y Home Assistant de manera que los valores causen poco daño incluso en caso de incidente.

La serie consta de tres partes: la [parte 1](/blog/midea-portasplit-home-assistant) describe la configuración hasta el panel de control; esta parte, la protección; y la [parte 3](/blog/midea-v2-cloud-api-portasplit-home-assistant) explica el contexto de las advertencias sobre la API en la nube.

![Panel de Home Assistant de la Midea PortaSplit en modo refrigeración: indicadores en la parte superior, termostato a 22 °C, gráficos de temperatura ambiente, consumo eléctrico, energía diaria, frecuencia del compresor, funcionamiento del compresor y velocidad del ventilador, seguidos de valores técnicos y estado.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

## De dónde proceden el token y la clave

En los dispositivos con protocolo V3, la PortaSplit solo acepta comandos locales con token y clave. Los valores no los genera el dispositivo, sino la nube de Midea; la aplicación oficial también los obtiene de allí. Las integraciones de la comunidad han reimplementado esta llamada a la nube: inician sesión en los mismos endpoints que la aplicación, reciben el token y la clave y guardan ambos localmente. Después, ya no se necesita conexión a la nube para el funcionamiento habitual.

No existe un mecanismo local de emparejamiento documentado que entregue los valores sin la nube. En teoría, podrían extraerse de la aplicación, por ejemplo mediante ingeniería inversa o instrumentación en tiempo de ejecución; para cada usuario esto es laborioso y no sustituye la obtención desde la nube. Por eso, si el endpoint deja de estar disponible, también desaparece la posibilidad de obtenerlos.

El proyecto `Midea AC LAN` advierte en su README que Midea podría cerrar gradualmente las interfaces de token en el futuro. Desde la revisión del README del 3 de septiembre de 2026, la integración obtiene el token y la clave a través de las nubes Meiju y SmartHome. Según el README, esto solo funciona con una cuenta personal a la que esté vinculado el dispositivo; actualmente no se pueden obtener tokens mediante NetHome Plus. Desde el 9 de septiembre de 2026, el README también indica que una cuenta SmartHome migrada desde otra aplicación no proporciona tokens; se necesita una cuenta recién registrada con dispositivos añadidos de nuevo. Los dispositivos ya configurados siguen funcionando localmente; se verían afectados los dispositivos nuevos y las nuevas configuraciones. Esto no constituye una hoja de ruta vinculante de Midea. En junio de 2026 también se comprobó que la supuestamente cerrada API de tokens de SmartHome seguía funcionando; la solicitud de la biblioteca de la comunidad simplemente estaba incompleta. La clasificación de la advertencia y de las distintas denominaciones «V2» se encuentra en la [parte 3](/blog/midea-v2-cloud-api-portasplit-home-assistant).

## Qué permiten el token y la clave

El token y la clave no tienen fecha de caducidad. Según `Midea AC LAN`, la comunicación del cliente se consideraba originalmente suficientemente protegida, por lo que la nube emitía tokens sin caducidad. Por sí mismo, esto no es una vulnerabilidad; se vuelve problemático cuando los valores terminan en registros o copias de seguridad sin protección, llegan a terceros o no pueden revocarse ni rotarse.

Quien posea el token y la clave y pueda acceder al dispositivo en la red puede autenticarse ante la PortaSplit, leer información de estado, encenderla y apagarla, cambiar modos de funcionamiento y modificar la temperatura objetivo. Los valores por sí solos no permiten un ataque desde Internet; el atacante necesita además una conexión de red con el dispositivo. Por ello, el token y la clave deben tratarse como una contraseña, y la red debería permitir esta conexión, en la medida de lo posible, solo a Home Assistant.

La integración de la comunidad no ataca el aire acondicionado. Implementa un protocolo propietario que se ha podido comprender mediante ingeniería inversa. El riesgo surge de que secretos de larga duración se almacenen fuera de la aplicación prevista.

## Proteger el token, la clave y la configuración

Guardar el token, la clave y la configuración es el paso único más importante: una vez cerradas las interfaces de token en la nube, una copia de seguridad será la única forma de realizar una nueva configuración. `Midea AC LAN` guarda un archivo de configuración JSON para dispositivos V3 tras una configuración correcta. La ruta documentada es:

```text
/config/.storage/midea_ac_lan/
```

El archivo lleva el ID del dispositivo como nombre:

```text
<device-id>.json
```

Este archivo no es una nota de texto normal. Puede contener el ID del dispositivo, el número de serie, la dirección IP, el token, la clave, información del protocolo, así como parámetros de la nube y del dispositivo. Por tanto:

- No lo suba a un repositorio público de GitHub.
- No lo publique en foros.
- No lo comparta como captura de pantalla sin ocultar datos.
- No lo envíe por correo electrónico sin cifrar.

Incluso un repositorio Git privado no es automáticamente el lugar adecuado, porque los secretos permanecen en el historial de Git aunque se eliminen posteriormente del archivo actual. Son más adecuados una copia de seguridad cifrada, un gestor de contraseñas con archivo adjunto, una copia de seguridad cifrada en NAS, un medio sin conexión cifrado o un archivo cifrado con la contraseña almacenada por separado.

Para hacer la copia de seguridad mediante el terminal de Home Assistant:

```bash
cd /config/.storage/midea_ac_lan
ls -la
```

Mostrar el archivo:

```bash
cat <device-id>.json
```

Para copiarlo, el archivo no debería transferirse mediante un servicio web público. Es mejor crear un archivo cifrado y trasladarlo después a una copia de seguridad cifrada:

```bash
tar -czf /config/midea-ac-lan-backup.tar.gz \
  /config/.storage/midea_ac_lan
```

Los archivos en `.storage` no deben editarse manualmente. El desarrollador recomienda expresamente no eliminar ni modificar directamente el archivo JSON en caso de problemas, sino renombrarlo y guardarlo antes de efectuar cambios.

Una copia de seguridad completa de Home Assistant también incluye estos archivos. Aun así, conviene disponer de una copia separada, ya que las copias de seguridad de Home Assistant pueden dañarse, una restauración puede sobrescribir la integración, el archivo podría necesitarse específicamente para una nueva configuración posterior y una copia de seguridad nunca debería estar solo en el mismo sistema.

### Eliminar secretos de un repositorio Git publicado

Si se ha publicado un archivo JSON por error en GitHub, no basta con eliminarlo normalmente y hacer un nuevo commit. El archivo sigue siendo accesible en el historial de Git. Como mínimo, se requieren estos pasos:

1. Poner el repositorio inmediatamente en privado, si es posible.
2. Eliminar el archivo de todo el historial de Git.
3. Tener en cuenta las cachés y los forks de GitHub.
4. Tratar el token como comprometido.
5. Eliminar el dispositivo de la cuenta de Midea y volver a conectarlo si eso genera nuevas claves.
6. Configurar de nuevo la integración de Home Assistant.
7. Cambiar la contraseña de la cuenta de Midea si las credenciales también se vieron afectadas.

Que el nuevo emparejamiento genere realmente un token nuevo varía según el dispositivo y la arquitectura de la nube. No debe confiarse en que cambiar la contraseña de la cuenta invalide automáticamente el token local del dispositivo.

## Aislar la PortaSplit en la red

### Sin redirección de puertos hacia la PortaSplit

El error evitable más frecuente sería hacer accesible directamente desde Internet el puerto local del dispositivo. Una regla como esta sería peligrosa:

```text
Internet → TCP 6444 → PortaSplit
```

No hay una buena razón para hacer accesible la PortaSplit directamente desde Internet. Home Assistant ya se encuentra en la red local y sirve como instancia de control. El router no debería tener ninguna redirección de puertos hacia la PortaSplit, debería restringir o desactivar UPnP cuando sea posible, bloquear por defecto las conexiones entrantes y no utilizar una asignación DMZ para el dispositivo.

### VLAN de IoT propia

La mejor arquitectura de red es una red de IoT separada:

```text
VLAN 10: vertrauenswürdige Clients
VLAN 20: Server und Home Assistant
VLAN 30: IoT-Geräte
VLAN 40: Gäste
```

La PortaSplit se encuentra en la VLAN de IoT. Home Assistant puede acceder específicamente al dispositivo, pero la PortaSplit no debe poder acceder libremente a PC, NAS y otros sistemas internos. Una posible lógica de cortafuegos:

```text
Home Assistant → PortaSplit: erlauben
PortaSplit → Home Assistant: etablierte Verbindungen erlauben
PortaSplit → interne Clients: blockieren
PortaSplit → NAS: blockieren
PortaSplit → Management-Netz: blockieren
Internet → PortaSplit: blockieren
```

Durante la primera configuración, el dispositivo necesita acceso a Internet para la nube de Midea. Tras completar correctamente la configuración local, se puede comprobar si es posible bloquear el acceso saliente a Internet. No debería aplicarse inmediatamente un bloqueo definitivo. Primero hay que verificar si el control local sigue funcionando, si el dispositivo continúa accesible tras reiniciarlo, si resiste un reinicio del router, si sigue respondiendo después de varios días, si aún se necesita la aplicación MSmartHome y si siguen ofreciéndose actualizaciones de firmware. Quien desee seguir utilizando la nube y las actualizaciones de firmware puede permitir temporalmente el acceso saliente a Internet y bloquearlo de nuevo después.

### La segmentación de red puede impedir el descubrimiento

La búsqueda automática de dispositivos suele basarse en tráfico broadcast o multicast, que normalmente no se enruta entre límites de VLAN. Por ello, Home Assistant quizá no encuentre automáticamente la PortaSplit aunque se permita una conexión IP normal.

En ese caso, ayuda configurar temporalmente la PortaSplit en la misma VLAN que Home Assistant, indicar manualmente la IP del dispositivo, utilizar una función de retransmisión de broadcast adecuada o definir reglas de cortafuegos específicas después de la configuración. Desde el punto de vista de la seguridad, la configuración manual suele ser incluso la mejor variante, ya que no es necesario permitir tráfico broadcast adicional entre las redes.

### Asignación DHCP estática

La PortaSplit debería disponer de una asignación DHCP fija en el router:

```text
PortaSplit → 192.168.30.25
```

Normalmente es preferible una reserva DHCP a una IP estática configurada en el dispositivo. Home Assistant encuentra el dispositivo de forma fiable, las reglas de cortafuegos pueden limitarse a una dirección fija, el análisis de errores se simplifica y la asignación se mantiene estable tras reinicios del router o del dispositivo. De este modo, una regla de cortafuegos puede formularse de forma muy restrictiva:

```text
Home-Assistant-IP → 192.168.30.25:6444/TCP
```

El puerto realmente necesario debe verificarse según la integración y el dispositivo propio.

## Proteger Home Assistant y las integraciones

### Home Assistant como ancla central de confianza

Quien controla la PortaSplit localmente traslada parte de la confianza de la nube de Midea a Home Assistant. Si Home Assistant se ve comprometido, un atacante podría controlar no solo el aire acondicionado, sino todo el hogar inteligente.

Por ello, Home Assistant debería actualizarse periódicamente, no publicarse mediante una redirección de puertos sin protección, protegerse con una contraseña fuerte y única, utilizar autenticación multifactor, crear copias de seguridad cifradas, contener únicamente los complementos necesarios y no permitir acceso SSH innecesario desde Internet. Para el acceso remoto, una VPN, Home Assistant Cloud o un proxy inverso correctamente configurado son mejores opciones que una simple redirección de puertos al puerto 8123.

### HACS y el riesgo de la cadena de suministro

`Midea Smart AC` y `Midea AC LAN` son integraciones personalizadas. Se ejecutan dentro de Home Assistant y, por tanto, tienen amplio acceso a su entorno de ejecución. Una integración maliciosa o comprometida podría, en teoría, leer datos de configuración, extraer secretos, establecer conexiones de red, escanear dispositivos de la red local, leer estados de otras entidades, transferir datos a sistemas externos y perjudicar la disponibilidad de Home Assistant.

Esto no significa que las integraciones mencionadas sean maliciosas. Ambos proyectos son públicamente visibles, se desarrollan activamente y tienen una comunidad visible. Sin embargo, el código abierto no es una garantía automática de seguridad. Antes de instalarlo, conviene al menos comprobar si el repositorio recibe mantenimiento activo, si hay lanzamientos regulares, cuántas personas contribuyen al código, si existen incidencias de seguridad abiertas, si han cambiado recientemente los responsables o propietarios del repositorio, si HACS apunta al repositorio esperado y si una actualización contiene cambios inusualmente grandes o inexplicables.

Las actualizaciones no deberían instalarse a ciegas justo después de su publicación. Especialmente en sistemas de hogar inteligente críticos para la seguridad, es aconsejable esperar unos días y revisar las notas de la versión y los problemas comunicados.

### Los registros de depuración contienen datos sensibles

Ante problemas, los proyectos de código abierto suelen solicitar registros de depuración. La documentación de `Midea AC LAN` muestra cómo activar el registro para los dos componentes relevantes:

```yaml
logger:
  default: warn
  logs:
    custom_components.midea_ac_lan: debug
    midealocal: debug
```

Después, los registros pueden descargarse desde Configuración, Sistema y Registros. Según la integración y el error, estos registros pueden contener direcciones IP locales, ID del dispositivo, número de serie, identificador de modelo, respuestas de la nube, información de cuenta, el token o partes de este, paquetes de red, así como marcas de tiempo y patrones de uso. Por ello, deben revisarse y ocultarse los valores sensibles antes de subirlos a una incidencia pública de GitHub.

Una vez terminada la búsqueda de errores, debe eliminarse de nuevo el registro de depuración. Mantenerlo activado permanentemente no solo incrementa el uso de almacenamiento, sino que también amplía la cantidad de información sensible en las copias de seguridad.

## Nube y firmware

### Proteger la cuenta en la nube

Mientras se utilice la nube de Midea para la configuración o el control mediante la aplicación, la cuenta de Midea sigue formando parte del modelo de seguridad. Debe tener una contraseña única que no se comparta con otros servicios, un gestor de contraseñas, autenticación multifactor si se ofrece, la eliminación de teléfonos antiguos y sesiones, evitar cuentas compartidas y revisar periódicamente qué dispositivos están registrados en la cuenta.

Si la integración de Home Assistant solicita nombre de usuario y contraseña durante la configuración, debe comprobarse si las credenciales se utilizan solo para la obtención única del token o si se almacenan permanentemente. Los desarrolladores de `Midea Smart AC` escriben que los dispositivos no se vinculan a cuentas integradas de la integración tras la configuración y que el token y la clave también pueden obtenerse manualmente mediante CLI con la cuenta propia. Siempre que sea posible, debe preferirse la cuenta propia a cuentas ajenas o colectivas integradas. En `Midea AC LAN`, desde septiembre de 2026 es requisito para obtener el token contar con una cuenta personal de Meiju o SmartHome a la que esté vinculado el dispositivo.

### ¿Bloquear la nube o no?

Tras completar correctamente la configuración, surge la pregunta de si se debería bloquear por completo el acceso a Internet de la PortaSplit. A favor del bloqueo están una menor telemetría, una menor dependencia de servicios externos, una menor superficie de ataque a través de la nube del fabricante, el hecho de que el dispositivo no pueda contactar con destinos externos arbitrarios y un menor impacto de los cambios en la nube.

En contra está que la aplicación MSmartHome podría dejar de funcionar fuera de la red doméstica, que las actualizaciones de firmware dejarían de descargarse, que podrían fallar funciones de hora o de nube, que un nuevo inicio de sesión o una restauración resultarían más difíciles y que algunos dispositivos pueden reaccionar de forma inesperada tras un periodo prolongado sin conexión.

Un orden pragmático: configurar el dispositivo con normalidad, probar Home Assistant y la aplicación, guardar el token y la configuración, bloquear el acceso a Internet, reiniciar el dispositivo y Home Assistant, observar durante varios días y, si es necesario, volver a permitir el acceso a Internet solo temporalmente.

### Actualizaciones de firmware: ¿ganancia de seguridad o riesgo para la integración?

Las actualizaciones de firmware son un dilema en los dispositivos IoT. Pueden cerrar vulnerabilidades conocidas, mejorar la estabilidad, modernizar mecanismos de seguridad y aportar nuevas funciones. Pero también pueden modificar interfaces locales, romper integraciones basadas en ingeniería inversa, invalidar tokens, desactivar la API local e introducir nuevas dependencias de la nube.

El firmware de PortaSplit distribuido en enero de 2026 incorporó, por ejemplo, un nuevo modo silencioso para la unidad exterior, que reduce el nivel de ruido en unos 6 decibelios. Las integraciones de la comunidad tuvieron que analizarlo e implementarlo primero, documentado en una incidencia propia de GitHub para la PortaSplit.

De ello se desprende que las actualizaciones de firmware no deben impedirse por principio: antes de actualizar, conviene comprobar si otros usuarios de Home Assistant informan de problemas, guardar previamente la configuración y el token, crear una copia de seguridad de Home Assistant y probar por completo el control local después de la actualización. Seguridad no significa «no actualizar nunca». Un firmware obsoleto puede ser más peligroso que una integración temporalmente incompatible.

### Lo que Midea dice sobre seguridad

Midea promociona su ecosistema SmartHome por su orientación a varios estándares de seguridad y protección de datos, entre ellos EN 303 645, UK PSTI, NIST, tratamiento de datos conforme al RGPD y los requisitos de la Directiva europea sobre equipos radioeléctricos. Son señales positivas, pero no indican cómo está implementado realmente cada firmware de PortaSplit, cada endpoint en la nube y cada API local. Las declaraciones de certificación y marketing no sustituyen una revisión técnica del dispositivo concreto.

Del mismo modo, sería erróneo deducir de la advertencia de una integración de la comunidad que la PortaSplit es insegura en general. El problema descrito afecta a la arquitectura de los tokens de larga duración y a su uso por clientes no oficiales.

## Riesgo por escenario

| Escenario | Riesgo | Justificación |
| --- | --- | --- |
| Red doméstica normal sin redirección de puertos | manejable | Un atacante necesita primero acceder a la Wi‑Fi, a Home Assistant o a una copia de seguridad. |
| Red doméstica plana con muchos dispositivos IoT inseguros | medio | Otro dispositivo IoT comprometido puede acceder a PortaSplit o a Home Assistant en la misma red. |
| PortaSplit accesible directamente desde Internet | alto | El dispositivo nunca debería publicarse mediante redirección de puertos. |
| Token y clave públicos en GitHub | alto | Los secretos se consideran comprometidos; no está garantizado que puedan revocarse. |
| VLAN de IoT separada, cortafuegos restrictivo y control local | comparativamente bajo | Incluso si existe una vulnerabilidad en el dispositivo, la libertad de movimiento en la red queda muy limitada. |

## Lista de comprobación

```text
1. Home-Assistant-Backup anfertigen
2. Token- und Konfigurationsdaten verschlüsselt sichern
3. DHCP-Reservation für die PortaSplit einrichten
4. Keine Portweiterleitung, UPnP einschränken
5. PortaSplit in ein separates IoT-VLAN verschieben
6. Zugriff von Home Assistant zur PortaSplit erlauben
7. Zugriff der PortaSplit auf interne Netze blockieren
8. Internetzugriff testweise blockieren
9. lokale Steuerung nach Neustarts prüfen
10. Firmware- und Integrationsupdates kontrolliert durchführen
```

La dirección de comunicación deseada:

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

Operado de este modo, el control local es aceptable desde el punto de vista de la seguridad: el token y la clave permanecen secretos y protegidos, el dispositivo solo es accesible para Home Assistant, y las actualizaciones de firmware e integración se aplican de forma controlada.

## Historial de cambios

- **25 de octubre de 2026:** Añadido: desde la revisión del README de septiembre de 2026, Midea AC LAN exige para obtener el token una cuenta personal de Meiju o SmartHome a la que esté vinculado el dispositivo; NetHome Plus y las cuentas SmartHome migradas no proporcionan tokens.

## Fuentes

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integración `Midea AC LAN` con el «Important Notice» (desde el 19 de mayo de 2025, revisado el 3 y el 9 de septiembre de 2026), la explicación sobre los tokens sin caducidad y la descripción de la obtención de tokens basada en la nube mediante una cuenta personal de Meiju o SmartHome.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integración `Midea Smart AC`: obtención de token y clave basada en la nube en dispositivos V3, almacenamiento local de los valores, puerto estándar 6444.

3.  [midea_ac_lan: indicaciones de depuración y configuración](https://github.com/wuwentao/midea_ac_lan/blob/main/doc/debug.md): almacenamiento de la configuración del dispositivo en `/config/.storage/midea_ac_lan/`, recomendación de guardar el archivo JSON en lugar de eliminarlo y configuración del registrador para registros de depuración.

4.  [Issue 779: modo silencioso exterior de PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/779): solicitud de soporte para el modo silencioso de la unidad exterior introducido con la actualización de firmware de enero de 2026, que reduce el nivel de ruido en unos 6 decibelios.

5.  [Midea SmartHome](https://www.midea.com/global/smarthome): información del fabricante sobre los estándares de seguridad y protección de datos EN 303 645, PSTI, NIST, RGPD y RED DA.

6.  [Home Assistant Community Store (HACS)](https://www.hacs.xyz/): instalación y gestión de integraciones personalizadas que no forman parte de Home Assistant Core.
