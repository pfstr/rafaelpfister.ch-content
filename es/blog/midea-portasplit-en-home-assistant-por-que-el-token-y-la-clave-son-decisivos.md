---
title: "Midea PortaSplit en Home Assistant: por qué el token y la clave son decisivos"
navTitle: "PortaSplit y token"
description: "El control local requiere dos valores de la nube de Midea. Cómo obtener el token y la clave, por qué su pérdida es problemática y cómo los propietarios pueden proteger su configuración actual."
date: "2026-07-24"
kategorie: "Home Assistant e IoT"
timeToRead: "9 min de lectura"
themen:
  - "smart-home-iot"
related:
  - "midea-portasplit-home-assistant-einrichten"
  - "serverloser-newsletter-cloudflare-workers-d1"
image: "../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png"
slug: "midea-portasplit-en-home-assistant-por-que-el-token-y-la-clave-son-decisivos"
translationOf: "midea-portasplit-home-assistant"
url: "https://rafaelpfister.ch/es/blog/midea-portasplit-en-home-assistant-por-que-el-token-y-la-clave-son-decisivos"
---

<aside class="article-update">
  <p class="article-update__label">Lo que los propietarios de PortaSplit deberían hacer ahora</p>
  <p>A través de interfaces privadas de la nube, Home Assistant obtiene el token y la clave específicos del dispositivo durante la configuración. El proyecto Midea AC LAN advierte desde el 19 de mayo de 2025 sobre posibles cambios. Sin embargo, no hay documentada una fecha concreta de desactivación por parte del fabricante. Para los propietarios, esto significa:</p>
  <ol>
    <li><strong>No deshacer innecesariamente una configuración existente.</strong> Solo la obtención de las credenciales requiere la nube de Midea. Futuras modificaciones del endpoint privado podrían dificultar una nueva configuración.</li>
    <li><strong>Guardar de forma cifrada el token, la clave y la configuración.</strong> Si la consulta deja de funcionar más adelante, la copia de seguridad seguirá siendo la forma más fiable de restaurar el sistema.</li>
    <li><strong>No deshacer la vinculación sin necesidad.</strong> El restablecimiento de fábrica, la eliminación de la cuenta de Midea o el cambio del módulo Wi-Fi obligan a obtener un nuevo token, algo que podría fallar en el futuro.</li>
  </ol>
  <p>Los dispositivos ya configurados se controlan localmente. Por ello, los cambios en la interfaz de la nube afectan primero a la incorporación y restauración, no a cada comando de control en funcionamiento. Los pasos concretos se explican en el <a href="/blog/midea-portasplit-home-assistant-einrichten">artículo práctico sobre integración y protección</a>.</p>
</aside>

![Panel de Home Assistant de ejemplo para una Midea PortaSplit con temperatura ambiente y objetivo, humedad, consumo eléctrico, consumo energético y tiempos de funcionamiento del compresor de las últimas 24 horas.](../images/midea-portasplit-home-assistant/home-assistant-dashboard-portasplit.png)

El control local de la Midea PortaSplit se basa en dos valores específicos del dispositivo: token y clave. La integración de Home Assistant recupera ambos durante la configuración a través de un endpoint privado de la nube de Midea. Después, envía los comandos de control directamente en la red local.

El proyecto Midea AC LAN advierte sobre posibles cambios en estas interfaces de la nube. Sin embargo, análisis más recientes muestran que de ello no puede deducirse una hoja de ruta confirmada del fabricante ni una fecha concreta de desactivación. Este artículo explica la relación de dependencia técnica; el [análisis detallado de la API](/blog/midea-v2-cloud-api-portasplit-home-assistant) contextualiza las distintas denominaciones «V2» y la situación actual.

## La cuestión del token en detalle

### ¿Por qué Home Assistant podía obtener hasta ahora el token?

Lo interesante es que la comunidad nunca calculó el token. En su lugar, analizó el tráfico de red de la aplicación oficial y comprobó que la aplicación no genera el token por sí misma, sino que lo obtiene de la nube:

```text
App
   ↓
Midea Cloud
   ↓
Cloud liefert Token
   ↓
App verwendet Token lokal
```

La integración de Home Assistant reimplementó precisamente esta llamada a la nube. Inicia sesión en la nube con los mismos endpoints y el mismo proceso que la aplicación, y así obtiene el mismo token y clave. Por tanto, la base real no es un cálculo ingenioso, sino una recuperación reproducida. Si el endpoint desaparece, también desaparece la posibilidad de obtenerlos.

### ¿Se podría extraer el token de la aplicación oficial?

Teóricamente sí. La aplicación debe conocer el token en algún momento; de lo contrario, no podría comunicarse localmente con el dispositivo. Algunas vías concebibles serían:

- ingeniería inversa de la aplicación,
- interceptar el tráfico de red, si no está protegido adicionalmente,
- instrumentar la aplicación en tiempo de ejecución, por ejemplo con Frida u Objection,
- hacer hooking de las funciones que procesan el token.

Precisamente a esto se refiere el desarrollador de Midea AC LAN cuando afirma que el diseño actual constituye un problema de seguridad desde la perspectiva de Midea: un secreto duradero que puede extraerse con un esfuerzo razonable de una aplicación ampliamente distribuida es difícil de controlar. Para cada usuario, sin embargo, estas vías son complejas y no sustituyen la cómoda recuperación desde la nube.

### ¿Se podría obtener el token directamente del dispositivo?

Esa sería la solución más elegante. Si el dispositivo intercambiara una clave pública en el primer emparejamiento local o utilizara un código de emparejamiento único por Bluetooth, no haría falta ninguna nube. Muchos dispositivos IoT modernos funcionan precisamente así.

Sin embargo, Midea diseñó de otra manera el protocolo LAN original: el dispositivo solo acepta comandos locales con las credenciales adecuadas relacionadas con la nube. No existe un mecanismo local de emparejamiento documentado que entregue el token sin pasar por la nube. Por tanto, la nube no es solo una comodidad, sino arquitectónicamente la única vía prevista para obtener el token.

### ¿Podría la comunidad sortear los cambios en el endpoint del token?

Solo sería posible si se encontrara una de las siguientes opciones:

- una nueva API de nube que siga proporcionando tokens,
- un método de emparejamiento local hasta ahora desconocido,
- una vulnerabilidad en el dispositivo,
- o que Midea publique algún día una API local oficial.

En cambio, simplemente «recalcular» el token probablemente no funcionará. Si fuera posible, la comunidad seguramente ya lo habría implementado hace tiempo y nunca habría dependido de la API de nube. Que se haya desarrollado el desvío a través de la nube es el indicio más sólido de que no existe una vía local más sencilla.

## La advertencia de Midea AC LAN

El repositorio de `Midea AC LAN` contiene un destacado aviso «Important Notice». Según el desarrollador, Midea ya ha cerrado las API de token del lado del servidor en las nubes Meiju y SmartHome. Por ello, la integración utiliza actualmente las interfaces de token de la nube NetHome Plus, que también se cerrarían gradualmente. La consecuencia sería que los dispositivos ya configurados seguirían funcionando localmente, pero ya no se podrían añadir dispositivos nuevos. El desarrollador va más allá y escribe que Midea pretende migrar a largo plazo a una nueva API de control en la nube, dejando inutilizable la anterior API LAN V1.

La advertencia tiene una breve historia. El destacado aviso «Important Notice» se añadió al README el 19 de mayo de 2025 (Pull Request #578) y entonces mencionaba la nube SmartHome como alternativa para añadir nuevos dispositivos. El 14 de julio de 2025 (#639) se actualizó; desde entonces remite a la nube NetHome Plus, porque Midea había cerrado más endpoints. El núcleo se mantuvo sin cambios en ambas versiones: las interfaces de token van desapareciendo gradualmente; solo cambia la nube que aún puede utilizarse en cada momento.

Esto debe considerarse con matices. Se trata de la valoración de un proyecto de código abierto, no de una hoja de ruta vinculante de Midea, y el calendario se desconoce. Una futura actualización de firmware puede modificar las funciones locales; un token ya guardado puede seguir funcionando, pero no necesariamente para siempre. Un restablecimiento de fábrica, el cambio del módulo Wi-Fi o un dispositivo nuevo pueden requerir obtener un token de nuevo.

De ello se derivan los tres pasos del recuadro al inicio del artículo, cada uno con su justificación:

- **No sustituir una configuración funcional sin motivo.** La obtención del token es el único paso que debe pasar obligatoriamente por la nube de Midea. Los cambios en el endpoint privado pueden afectar sobre todo a una configuración posterior.
- **Guardar las credenciales.** Home Assistant almacena localmente el token y la clave. Un sistema averiado, una restauración fallida o una integración eliminada por accidente pueden dejar inservible el control local si no existe una copia de seguridad externa.
- **No deshacer la vinculación a la ligera.** No está completamente documentado si un restablecimiento de fábrica o la eliminación de la cuenta de Midea obliga a obtener nuevas credenciales en todos los modelos. Por ello, es imprescindible hacer una copia de seguridad antes de estos cambios.

El funcionamiento diario no se ve afectado inicialmente: el control local utiliza los valores ya almacenados y ya no necesita el endpoint del token. Sigue existiendo un riesgo residual si un firmware posterior cambia el protocolo local o la autenticación. Cómo guardar el token, la clave y la configuración se explica en el [artículo práctico sobre la configuración](/blog/midea-portasplit-home-assistant-einrichten#backup-der-konfiguration).

## Qué significa esto para la seguridad

Además de la disponibilidad, la advertencia tiene un componente central de seguridad. Según `Midea AC LAN`, la arquitectura LAN más antigua se basa en una premisa problemática: originalmente se consideraba que la comunicación del cliente estaba suficientemente protegida, por lo que los tokens emitidos por la nube no tenían fecha de caducidad.

Un token que no caduca no constituye por sí solo una vulnerabilidad. Se vuelve problemático si acaba en registros o copias de seguridad sin protección, llega a terceros o no puede revocarse ni rotarse. El desarrollador de `Midea AC LAN` supone que Midea está respondiendo a estos riesgos con cambios en los servicios de token y una arquitectura más basada en la nube. Sin embargo, no hay constancia de un anuncio correspondiente del fabricante con un calendario.

Aquí es importante la precisión lingüística. La integración comunitaria no «hackea» el aire acondicionado. Implementa un protocolo propietario que se ha comprendido mediante ingeniería inversa. El problema de seguridad surge porque secretos duraderos pueden utilizarse y almacenarse fuera de la aplicación originalmente prevista.

Para el funcionamiento en la propia red, lo más relevante es qué permiten el token y la clave. Ambos autentican la comunicación local con el dispositivo. Si caen en malas manos, un atacante podría, dependiendo del protocolo y de su posición en la red, detectar el dispositivo, autenticarse ante él, leer información de estado, modificar ajustes, encender o apagar el aire acondicionado, cambiar los modos de funcionamiento y modificar la temperatura objetivo. Por regla general, el atacante debe poder establecer una conexión de red con el dispositivo; poseer únicamente el token y la clave no permite atacar desde cualquier lugar de Internet. Por ello, el token y la clave deben tratarse como una contraseña. Cómo integrar el dispositivo en la red para que estos valores causen poco daño incluso si ocurre un incidente es el tema de la [segunda parte](/blog/midea-portasplit-home-assistant-einrichten#die-portasplit-sicher-betreiben).

## Qué queda en la práctica

El control local de la PortaSplit depende por completo del token y la clave, que actualmente solo pueden obtenerse a través de la nube de Midea. Este desvío forma parte del diseño del protocolo: los comandos locales están vinculados a credenciales relacionadas con la nube. Puesto que el endpoint es privado y no está documentado, la disponibilidad a largo plazo de la integración no oficial sigue siendo incierta.

En la práctica, esto significa: guardar las credenciales y la configuración, no deshacer innecesariamente una vinculación funcional y vigilar los cambios en la integración y el firmware. Los dispositivos ya configurados siguen funcionando localmente. El [artículo práctico sobre PortaSplit](/blog/midea-portasplit-home-assistant-einrichten) describe la configuración, las copias de seguridad y la protección de red.

## Fuentes

1.  <a class="gh-badge" href="https://github.com/wuwentao/midea_ac_lan" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">wuwentao/midea_ac_lan</span></a>: Integración `Midea AC LAN` con el aviso «Important Notice» (desde el 19 de mayo de 2025, actualizado el 14 de julio de 2025), la justificación basada en tokens sin caducidad y cifrado de cliente reconstruido, así como la descripción de la obtención del token basada en la nube.

2.  <a class="gh-badge" href="https://github.com/mill1000/midea-ac-py" rel="noopener"><span class="gh-badge__label"><svg width="13" height="13" viewBox="0 0 16 16" fill="currentColor" aria-hidden="true"><path d="M8 0C3.58 0 0 3.58 0 8c0 3.54 2.29 6.53 5.47 7.59.4.07.55-.17.55-.38 0-.19-.01-.82-.01-1.49-2.01.37-2.53-.49-2.69-.94-.09-.23-.48-.94-.82-1.13-.28-.15-.68-.52-.01-.53.63-.01 1.08.58 1.23.82.72 1.21 1.87.87 2.33.66.07-.52.28-.87.51-1.07-1.78-.2-3.64-.89-3.64-3.95 0-.87.31-1.59.82-2.15-.08-.2-.36-1.02.08-2.12 0 0 .67-.21 2.2.82.64-.18 1.32-.27 2-.27s1.36.09 2 .27c1.53-1.04 2.2-.82 2.2-.82.44 1.1.16 1.92.08 2.12.51.56.82 1.27.82 2.15 0 3.07-1.87 3.75-3.65 3.95.29.25.54.73.54 1.48 0 1.07-.01 1.93-.01 2.2 0 .21.15.46.55.38A8.01 8.01 0 0 0 16 8c0-4.42-3.58-8-8-8Z"/></svg>GitHub</span><span class="gh-badge__name">mill1000/midea-ac-py</span></a>: Integración `Midea Smart AC`: descripción de la obtención del token y la clave basada en la nube para dispositivos V3 y del almacenamiento local de los valores.

3.  [Midea SmartHome](https://www.midea.com/global/smarthome): información del fabricante sobre el ecosistema SmartHome y los estándares de seguridad y privacidad citados.
