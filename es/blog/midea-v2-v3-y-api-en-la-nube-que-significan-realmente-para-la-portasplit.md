---
title: "Midea V2, V3 y API en la nube: qué significa realmente para la PortaSplit"
navTitle: "API en la nube Midea V2"
description: "El protocolo local del dispositivo, los endpoints privados de la aplicación y la API oficial para socios usan nombres de versión similares. El análisis de las fuentes separa estos niveles y contextualiza la advertencia de desactivación."
date: "2026-07-25"
kategorie: "Home Assistant e IoT"
timeToRead: "11 min de lectura"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant-absichern
  - midea-portasplit-home-assistant
draft: false
slug: "midea-v2-v3-y-api-en-la-nube-que-significan-realmente-para-la-portasplit"
translationOf: "midea-v2-cloud-api-portasplit-home-assistant"
translationId: article-f504b2af00493864
translationModel: gpt-5.6-terra
translatedAt: 2026-11-06T03:20:04.400Z
translationReview: automatic
translationSourceHash: 551c9e10e7e9c6a84b6e439dd388d775dae5246f96cd6c71e46c6065e84c5fd7
image: ../images/midea-portasplit-home-assistant/portasplit-dashboard.png
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/midea-v2-v3-y-api-en-la-nube-que-significan-realmente-para-la-portasplit
---

En el entorno de la Midea PortaSplit, «V2» designa varias cosas independientes entre sí. Existe un protocolo local V2 del dispositivo, números de versión en endpoints privados de la aplicación y una API V2 oficial de nube a nube para socios. Quien equipare estos niveles llegará inevitablemente a conclusiones erróneas sobre el control local.

El proyecto `Midea AC LAN` advierte en su [README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) de que las interfaces de tokens utilizadas hasta ahora se cerrarían y serían sustituidas por una API V2 basada en la nube. Una revisión de los debates, del código actual y de la documentación oficial de Midea ofrece una imagen más matizada:

> Existe una API V2 oficial de Midea de nube a nube. Sin embargo, no es idéntica a la interfaz de tokens utilizada por Home Assistant ni al protocolo local V2 o V3 del dispositivo. No está documentada una desactivación anunciada oficialmente del control local de la PortaSplit con una fecha concreta. Además, en junio de 2026 se demostró que la API de tokens de SmartHome supuestamente desactivada seguía funcionando: la solicitud anterior de la biblioteca de la comunidad simplemente estaba incompleta.

Esta es la parte 3 de la serie; la [parte 1](/blog/midea-portasplit-home-assistant) describe la configuración hasta el panel y la [parte 2](/blog/midea-portasplit-home-assistant-absichern) la protección del token, la clave y la red doméstica. Este artículo está actualizado al 25 de octubre de 2026.

![Panel de Home Assistant de la Midea PortaSplit en modo refrigeración: indicadores en la parte superior, termostato a 22 °C, históricos de temperatura ambiente, consumo eléctrico, energía diaria, frecuencia del compresor, funcionamiento del compresor y nivel del ventilador; debajo, valores técnicos y estado.](../images/midea-portasplit-home-assistant/portasplit-dashboard.png)

## Por qué debe corregirse la clasificación anterior

En una versión anterior del [artículo sobre el token y la clave](/blog/midea-portasplit-home-assistant-absichern) había reproducido, en esencia, la advertencia del proyecto `Midea AC LAN` como una desactivación anunciada de las interfaces en la nube. Esto correspondía al texto de la README del proyecto, pero estaba formulado de forma demasiado categórica como una afirmación de hecho.

La advertencia sigue siendo relevante como indicación de riesgo. Sin embargo, no es una hoja de ruta publicada por Midea. Sobre todo, ahora hay disponible nuevo material técnico que cuestiona una parte esencial de la interpretación anterior.

## Cómo funciona el control local de la PortaSplit

La integración de Home Assistant `Midea Smart AC` describe explícitamente su arquitectura como control local. En los dispositivos V3 más recientes, la nube de Midea se utiliza únicamente durante la configuración para obtener un token y una clave específicos del dispositivo. Después, la integración guarda ambos valores localmente y no necesita ninguna conexión adicional a la nube para el control propiamente dicho. El proyecto lo documenta en [«Note On Cloud Usage»](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

De forma simplificada, el proceso es el siguiente:

```text
Einrichtung:

Home Assistant
    │
    ├── Anmeldung an einer Midea-Cloud
    ├── Abruf von Geräte-ID, Token und Key
    └── lokale Speicherung der Zugangsdaten

Normalbetrieb:

Home Assistant
    │
    └── lokale TCP-Verbindung zur PortaSplit
```

Para los dispositivos V3 configurados manualmente, `Midea Smart AC` requiere el ID del dispositivo, la dirección IP, el puerto, el token y la clave. El puerto estándar documentado es `6444/TCP`; el token y la clave se indican como 128 y 64 caracteres hexadecimales, respectivamente. Estos datos figuran en la [documentación de configuración manual](https://github.com/mill1000/midea-ac-py#manual-configuration).

Por ejemplo, en el rastreador de incidencias de `Midea AC LAN` se detectó una PortaSplit como tipo de dispositivo `0xAC`, modelo `00000Q1D` y versión de protocolo 3. Posteriormente, el mismo usuario pudo añadirla a Home Assistant mediante NetHome Plus. El proceso concreto está documentado en [Issue #607](https://github.com/wuwentao/midea_ac_lan/issues/607).

La separación es decisiva:

- El servicio en la nube se utiliza para obtener las credenciales locales.
- El control posterior se realiza directamente en la LAN.
- Por tanto, una interrupción del servicio de tokens afecta principalmente a las nuevas configuraciones.
- No finaliza automáticamente una conexión local ya configurada.

Esto último también corresponde a la descripción expresa de [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## De dónde procede la advertencia de desactivación

El texto de advertencia visible actualmente se incorporó a la documentación el 19 de mayo de 2025 mediante [Pull Request #578](https://github.com/wuwentao/midea_ac_lan/pull/578).

En resumen, la justificación es la siguiente:

- Los tokens locales no tendrían fecha de caducidad.
- Varios proyectos de Home Assistant utilizarían cifrado de la aplicación reproducido o extraído.
- De ello se derivaría un problema de seguridad.
- Por este motivo, Midea cerraría gradualmente los servicios de tokens existentes.
- A largo plazo, el control local V1 sería desplazado por una API V2 basada en la nube.

En julio de 2025, la documentación se volvió a adaptar mediante [Pull Request #639](https://github.com/wuwentao/midea_ac_lan/pull/639). En lugar de la nube SmartHome, se mencionó entonces NetHome Plus como fuente de tokens utilizada temporalmente. La advertencia de desactivación propiamente dicha se mantuvo.

Sin embargo, el debate subyacente está formulado con más cautela que la README.

En el [comentario del mantenedor de Midea AC LAN](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457) se afirma, en esencia, que NetHome Plus podría ser solo una solución temporal y que, según su interpretación, Midea dispone de un nuevo servicio V2 totalmente basado en la nube.

El mantenedor de `midea-msmart` respondió que también había supuesto la existencia de una nueva API V2, pero que, al no disponer de dispositivos Midea propios, solo podía investigarla de forma limitada. Esto consta en el [comentario de respuesta directa](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

Con ello, la situación de las fuentes queda más clara:

- La advertencia procede de desarrolladores experimentados de la comunidad.
- Se basa en cambios observados y en su valoración técnica.
- Uno de los mantenedores califica expresamente la migración a V2 como su interpretación.
- El otro habla de una suposición.
- Ni el pull request ni el debate enlazan un anuncio oficial de desactivación de Midea o una fecha.

Esto no hace que la advertencia carezca de valor. Pero la convierte en un análisis de riesgos, no en una hoja de ruta confirmada por el fabricante.

## El nuevo hallazgo decisivo de junio de 2026

El 15 de junio de 2026 se integró en la biblioteca `midea-local` una corrección que modifica sustancialmente la interpretación anterior.

El punto de partida fue el error:

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

Este error se produjo al consultar el token y la clave mediante la nube SmartHome. El inicio de sesión y la lista de dispositivos seguían funcionando, pero se rechazaba la llamada a `/v1/iot/secure/getToken`.

Al principio, esto parecía una interfaz desactivada o inutilizada. Sin embargo, un análisis de la solicitud de la aplicación oficial SmartHome mostró otra causa: además de `udpid`, la aplicación enviaba el campo `applianceCodes`. La biblioteca de la comunidad no enviaba este campo.

La solicitud corregida contiene ahora:

```python
data.update({
    "udpid": udp_id,
    "applianceCodes": str(appliance_id)
})
```

El desarrollador probó el cambio con una cuenta real de SmartHome y cuatro equipos de aire acondicionado V3 del tipo `0xAC`:

- Sin `applianceCodes`, el servidor respondía con el error 3004.
- Con `applianceCodes`, proporcionaba tokens y claves válidos.
- Los valores devueltos funcionaron posteriormente para la autenticación local V3.

La investigación completa, los resultados de las pruebas y el diff del código están documentados en [`midea-local` Pull Request #470](https://github.com/midea-lan/midea-local/pull/470). El commit inmutable asociado es [`23312799`](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5).

El código fuente actual sigue utilizando exactamente este endpoint:

```text
/v1/iot/secure/getToken
```

Además, ahora también se envía `applianceCodes`. Esto se puede comprobar directamente en el [código actual de `midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py).

Desde el 12 de agosto de 2026, `Midea AC LAN` utiliza su propia biblioteca `midea-lan` en lugar de `midea-local`; la versión actual incorpora `midea-lan==2026.10.0` y sigue declarándose como integración `local_push`. Ambas cosas constan en el [`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json) actual. También `midea-lan` consulta el token y la clave en la nube SmartHome a través de `/v1/iot/secure/getToken` con `applianceCodes`, como puede comprobarse en el [código de `midealan/cloud.py`](https://github.com/wuwentao/midea-lan/blob/main/midealan/cloud.py).

La afirmación general de que se ha cerrado la API de tokens SmartHome queda así refutada, al menos para las cuentas y dispositivos probados en junio de 2026. Lo correcto sería:

> La consulta anterior del token dejó de funcionar tras un cambio en el formato de solicitud esperado. Tras adaptarse al formato empleado por la aplicación oficial, el mismo endpoint V1 volvió a proporcionar credenciales locales válidas.

Esto no excluye diferencias regionales, cuentas distintas o tipos de dispositivos no compatibles. Pero evidentemente no fue una desactivación global.

Desde entonces, la situación ha cambiado en otras nubes de Midea. Desde agosto de 2026, la nube china Meiju responde a `/v1/iot/secure/getToken` con el error `40404`; allí, el token y la clave solo los proporciona ya el endpoint de la aplicación `/v2/iot/secure/getToken`, que además requiere un `homegroupId`. Las bibliotecas se adaptaron en consecuencia a finales de agosto de 2026, como se documenta en [`midea-local` Pull Request #717](https://github.com/midea-lan/midea-local/pull/717). Según la [README de `Midea AC LAN`](https://github.com/wuwentao/midea_ac_lan#1-important-notice), actualmente ya no se pueden obtener tokens mediante cuentas de NetHome Plus; solo pueden utilizarse cuentas personales de Meiju y SmartHome a las que esté vinculado el dispositivo. Este cambio también afecta a un endpoint privado de la aplicación, no a la API V2 oficial de nube a nube.

## Por qué «V2» se malinterpreta tan fácilmente aquí

En el entorno de Midea se utilizan al menos tres denominaciones de versión independientes entre sí.

| Término | Significado |
| --- | --- |
| Protocolo local V2/V3 | Generación de la comunicación directa entre la integración y el dispositivo |
| Endpoint de aplicación V1/V2 | Número de versión de un endpoint HTTP individual en el backend de las aplicaciones de Midea |
| API V2 de nube a nube | API oficial para socios y terceros autorizados |

### V2 y V3 locales

En el protocolo local del dispositivo, V2 y V3 designan la generación de comunicación del dispositivo. Los dispositivos V3 más recientes necesitan token y clave para la autenticación local. `Midea Smart AC` documenta este requisito en su [guía de configuración](https://github.com/mill1000/midea-ac-py#manual-configuration).

Esta versión de protocolo no tiene nada que ver con la API V2 oficial de nube a nube.

### V1 y V2 en las URL de la aplicación

Incluso dentro de la misma aplicación pueden utilizarse simultáneamente endpoints con diferentes números de versión. Por tanto, un `/v2/` en la ruta URL no significa que toda la plataforma se haya migrado a una arquitectura nueva.

El código actual de `midea-local` sigue utilizando [`/v1/iot/secure/getToken`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) para el token y la clave. Aun así, otras funciones pueden estar en rutas con versiones diferentes.

### API V2 oficial de nube a nube

Midea documenta efectivamente una [API V2 oficial de nube a nube](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Esta utiliza, entre otros elementos:

- OAuth 2.0
- `client_id` y `client_secret`
- tokens de acceso y tokens de actualización de corta duración
- firmas HMAC-SHA256
- `/v2/open/oauth2/authorize`
- `/v2/open/oauth2/token`
- `/v2/open/device/list/get`
- consultas de estado y comandos de control basados en la nube

Se trata de una interfaz controlada para socios. El `client_secret` necesario es asignado por Midea a un proveedor externo. Un propietario normal de una PortaSplit no lo obtiene simplemente con su cuenta de MSmartHome. Los requisitos y las reglas de firma se describen en la [documentación oficial de V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Además, esta API no nació en 2025. La documentación contiene ejemplos de solicitudes con marcas de tiempo de 2018 y un comentario de Java del 18 de abril de 2019. Por tanto, la interfaz V2 para socios ya existía mucho antes de la advertencia en `Midea AC LAN`.

## Midea sí sustituye una API V1, pero es otra

Midea también mantiene una interfaz oficial de nube a nube más antigua bajo `/v1/open/...`. Su documentación indica expresamente que ya no se recomienda, que podría desactivarse en el futuro y que debe utilizarse la nueva documentación V2. Esto figura en la [documentación de Midea sobre la antigua API de nube a nube](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html).

Esta indicación es una migración oficial real de V1 a V2. Sin embargo, afecta a los endpoints para socios:

```text
/v1/open/...
           ↓
/v2/open/...
```

En cambio, la consulta de tokens utilizada por las bibliotecas de Home Assistant es:

```text
/v1/iot/secure/getToken
```

Y la conexión local de la PortaSplit ya no pasa posteriormente por una URL en la nube de este tipo, sino directamente por la red doméstica.

Por tanto, equiparar las tres interfaces solo por el número de versión «V1» no estaría técnicamente justificado.

## ¿Ya existe una integración de Home Assistant totalmente basada en la nube?

Con [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud) existe ya una integración de la comunidad que controla dispositivos Midea a través de la nube en lugar de directamente por la LAN.

Sin embargo, tampoco esto demuestra que la API V2 oficial para socios ya haya sustituido al control local. El código fuente actual de `Midea Auto Cloud` utiliza, entre otros:

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

Estos endpoints se pueden consultar en el [código actual de `core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py).

Por tanto, la integración reproduce funciones privadas de la aplicación o de la nube de consumo. No utiliza simplemente la interfaz para socios documentada `/v2/open/...`.

Ya existe una alternativa basada en la nube. Pero también conlleva las dependencias habituales de una integración en la nube: acceso a Internet, una cuenta de usuario funcional, servidores de Midea disponibles y endpoints privados que sigan siendo compatibles.

## ¿Qué significa esto concretamente para los propietarios de una PortaSplit?

### Control local ya configurado

Para una PortaSplit ya configurada, la situación es relativamente poco problemática. `Midea Smart AC` guarda el token y la clave localmente tras la configuración y, según su propia [documentación sobre la nube](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage), no necesita conexión a la nube para el control posterior.

Por ello, la desactivación de la mera consulta de tokens no pondría fin automáticamente a la conexión local existente.

### Nueva configuración o restauración

El riesgo es mayor en caso de:

- una nueva instalación de Home Assistant
- cambiar a otra integración
- una copia de seguridad perdida o dañada
- sustituir el módulo Wi-Fi
- cambios en la asignación del dispositivo
- un nuevo emparejamiento, si con ello cambian las credenciales del dispositivo

En estos casos, la integración debe volver a obtener el token y la clave, o el usuario debe indicarlos manualmente. Que `Midea Smart AC` admite una configuración manual se describe en su [documentación de configuración](https://github.com/mill1000/midea-ac-py#manual-configuration).

No está documentado oficialmente si un restablecimiento de fábrica o un nuevo emparejamiento genera obligatoriamente nuevas credenciales en cada PortaSplit; por ello, no debería afirmarse de forma general.

### Una desactivación real del control por LAN

Para que una PortaSplit ya configurada dejara de aceptar sus credenciales guardadas localmente, tendría que cambiar además el comportamiento del dispositivo o del módulo Wi-Fi, por ejemplo mediante un nuevo firmware o un procedimiento de autenticación modificado.

La mera desactivación del endpoint en la nube `/v1/iot/secure/getToken` no elimina automáticamente las credenciales ya presentes en el dispositivo y en Home Assistant. Esto se deduce de la separación entre la consulta única en la nube y el control posterior por LAN documentada por [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

Este tipo de cambio futuro del dispositivo es técnicamente posible. Sin embargo, no he encontrado en la documentación pública de Midea un anuncio concreto ni una fecha de desactivación específicamente para la PortaSplit.

## Lo que seguiría recomendando

A pesar de los hallazgos que relativizan la situación, sigue siendo conveniente disponer de una copia de seguridad.

Para los dispositivos V3, `Midea AC LAN` recomienda expresamente guardar fuera de HAOS la configuración JSON generada. La recomendación actual figura directamente en la [README del proyecto](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

Una copia de seguridad es una protección razonable frente a cambios en la nube, problemas de integración y errores propios, pero no indica que una desactivación sea inminente. La [parte 2](/blog/midea-portasplit-home-assistant-absichern#token-key-und-konfiguration-sichern) explica cómo guardar el token, la clave y la configuración.

## Valoración basada en las pruebas disponibles

La advertencia de `Midea AC LAN` debe tomarse en serio, pero situarse correctamente en contexto.

Documenta un riesgo plausible a largo plazo: Midea podría considerar los tokens locales sin caducidad un problema de seguridad, restringir aún más la obtención de estos tokens o vincular más estrechamente los dispositivos futuros a la nube.

En cambio, no está probada una desactivación oficialmente anunciada y programada del control local de la PortaSplit.

El estado técnico actual muestra incluso lo contrario de una desactivación ya realizada: en junio de 2026, el endpoint V1 de tokens que seguía utilizándose proporcionó credenciales válidas después de adaptar la solicitud al formato de la aplicación oficial SmartHome. La corrección correspondiente forma hoy parte de la biblioteca utilizada por `Midea AC LAN`.

También existe la API V2 oficial de Midea de nube a nube. Sin embargo, se trata de una interfaz para socios más antigua y de acceso restringido, no automáticamente del sucesor del protocolo local de la PortaSplit.

Por ello, la conclusión sobria es:

> Cree una copia de seguridad, observe las integraciones y tenga presentes las dependencias de la nube, pero no descarte prematuramente el control local de la PortaSplit basándose en una suposición de desactivación no confirmada.

## Historial de cambios

- **25 de octubre de 2026:** Añadido: desde agosto de 2026, Midea AC LAN utiliza la biblioteca midea-lan (actualmente 2026.10.0), y la nube Meiju ya solo proporciona token y clave mediante el endpoint /v2/iot/secure/getToken con homegroupId. Se actualizó el artículo al 25 de octubre de 2026 y se añadieron fuentes.

## Fuentes

1.  [Midea AC LAN: README actual y advertencia de desactivación](https://github.com/wuwentao/midea_ac_lan#1-important-notice): texto de la advertencia, recomendación de copia de seguridad y distinción entre dispositivos V2 más antiguos y V3 más recientes.

2.  [Midea AC LAN PR #578 del 19 de mayo de 2025](https://github.com/wuwentao/midea_ac_lan/pull/578): introducción de la advertencia sobre la desactivación gradual de los servicios de tokens y la supuesta migración a una API V2 basada en la nube.

3.  [Midea AC LAN PR #639](https://github.com/wuwentao/midea_ac_lan/pull/639): cambio de la fuente de tokens documentada a NetHome Plus.

4.  [midea-msmart Issue #201](https://github.com/mill1000/midea-msmart/issues/201): debate sobre la consulta defectuosa de tokens de SmartHome y el uso temporal de NetHome Plus.

5.  [Comentario del mantenedor de Midea AC LAN sobre la supuesta migración a V2](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457): identifica expresamente la afirmación sobre la nueva nube V2 como su propia interpretación.

6.  [Respuesta del mantenedor de midea-msmart](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109): describe la existencia de una nueva API V2 como una suposición y señala las limitadas posibilidades de ingeniería inversa.

7.  [midea-local PR #470 del 15 de junio de 2026](https://github.com/midea-lan/midea-local/pull/470): análisis del error 3004, captura de la solicitud de la aplicación oficial, incorporación de `applianceCodes` y prueba satisfactoria con cuatro equipos de aire acondicionado V3.

8.  [Commit inmutable de la corrección de SmartHome-getToken](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5): diff exacto del código de la corrección incorporada.

9.  [Código actual de nube de midea-local](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py): endpoint `/v1/iot/secure/getToken` aún utilizado y campo de solicitud actual `applianceCodes`.

10.  [Manifiesto actual de Midea AC LAN](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json): versión utilizada de `midea-lan` y clasificación como integración de inserción local.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py): documentación del control local, de la consulta única en la nube para dispositivos V3 y de la configuración manual con token y clave.

12.  [Midea AC LAN Issue #607 sobre la PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607): ejemplo concreto de PortaSplit con tipo de dispositivo `0xAC`, modelo `00000Q1D`, versión de protocolo 3 y configuración satisfactoria mediante NetHome Plus.

13.  [API V2 oficial de Midea de nube a nube](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html): OAuth2, ID de cliente, secreto de cliente, tokens de acceso y de actualización, procedimiento de firma y endpoints `/v2/open/...`.

14.  [API V1 oficial de Midea de nube a nube](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html): aviso oficial de que la antigua interfaz para socios `/v1/open/...` ya no se recomienda y podría desactivarse en el futuro.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) y [código actual de nube](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py): integración de la comunidad para control completo en la nube y los endpoints privados V1 de la aplicación utilizados realmente.

16.  [midea-local PR #717](https://github.com/midea-lan/midea-local/pull/717): cambio de la nube Meiju de `/v1/iot/secure/getToken` (error 40404) a `/v2/iot/secure/getToken` con `homegroupId`.

17.  [Código actual de nube de midea-lan](https://github.com/wuwentao/midea-lan/blob/main/midealan/cloud.py): biblioteca utilizada por Midea AC LAN desde agosto de 2026, consulta de tokens de SmartHome mediante `/v1/iot/secure/getToken` con `applianceCodes`.
