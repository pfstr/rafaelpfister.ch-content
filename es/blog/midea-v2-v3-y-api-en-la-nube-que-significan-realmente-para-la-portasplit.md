---
title: "Midea V2, V3 y API en la nube: qué significan realmente para la PortaSplit"
navTitle: "API en la nube Midea V2"
description: "El protocolo local del dispositivo, los endpoints privados de la aplicación y la API oficial para socios usan nombres de versión similares. El análisis de las fuentes separa estos niveles y contextualiza la advertencia de desactivación."
date: "2026-07-25"
kategorie: "Home Assistant e IoT"
timeToRead: "11 min de lectura"
themen:
  - smart-home-iot
related:
  - midea-portasplit-home-assistant
  - midea-portasplit-home-assistant-einrichten
draft: false
slug: "midea-v2-v3-y-api-en-la-nube-que-significan-realmente-para-la-portasplit"
translationOf: "midea-v2-cloud-api-portasplit-home-assistant"
translationId: article-f504b2af00493864
translationModel: gpt-5.6-terra
translatedAt: 2026-09-04T08:25:44.778Z
translationReview: automatic
translationSourceHash: 12ce029c1de367a718159f3729a8d063f8c7df3982e1a0efa10be83a2af3b3ff
url: https://rafaelpfister.ch/es/blog/midea-v2-v3-y-api-en-la-nube-que-significan-realmente-para-la-portasplit
---

En el entorno de la Midea PortaSplit, «V2» designa varias cosas independientes entre sí. Existe un protocolo local de dispositivos V2, números de versión en endpoints privados de aplicaciones y una API V2 oficial de nube a nube para socios. Quien equipare estos niveles llegará inevitablemente a conclusiones erróneas sobre el control local.

El proyecto `Midea AC LAN` advierte en su [README](https://github.com/wuwentao/midea_ac_lan#1-important-notice) de que las interfaces de token utilizadas hasta ahora se cerrarían y serían sustituidas por una API V2 basada en la nube. Una revisión de los debates, del código actual y de la documentación oficial de Midea ofrece una imagen más matizada:

> Existe una API V2 oficial de Midea de nube a nube. Sin embargo, no es idéntica a la interfaz de token utilizada por Home Assistant ni al protocolo local de dispositivos V2 o V3. No hay documentación de un cierre oficialmente anunciado del control local de PortaSplit con una fecha concreta. Además, en junio de 2026 se demostró que la API de tokens de SmartHome supuestamente desactivada seguía funcionando: la solicitud anterior de la biblioteca de la comunidad simplemente estaba incompleta.

Este artículo está actualizado al 25 de julio de 2026.

## Por qué debe corregirse la clasificación anterior

En el [primer artículo sobre la cuestión de los tokens de nube](/blog/midea-portasplit-home-assistant) reproduje la advertencia del proyecto `Midea AC LAN` como si anunciara el cierre de las interfaces en la nube. Eso correspondía al texto de la README del proyecto, pero estaba formulado de forma demasiado contundente como afirmación de hecho.

La advertencia sigue siendo relevante como indicación de riesgo. Sin embargo, no es una hoja de ruta publicada por Midea. Sobre todo, ahora hay nuevo material técnico disponible que cuestiona una parte esencial de la interpretación anterior.

## Cómo funciona el control local de PortaSplit

La integración de Home Assistant `Midea Smart AC` describe explícitamente su arquitectura como control local. En los dispositivos V3 más recientes, la nube de Midea solo se utiliza durante la configuración para obtener un token y una clave específicos del dispositivo. Después, la integración guarda ambos valores localmente y no necesita ninguna otra conexión a la nube para el control propiamente dicho. El proyecto lo documenta en [«Note On Cloud Usage»](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

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

Para los dispositivos V3 configurados manualmente, `Midea Smart AC` requiere ID del dispositivo, dirección IP, puerto, token y clave. El puerto estándar documentado es `6444/TCP`; el token y la clave se indican como 128 y 64 caracteres hexadecimales, respectivamente. Esta información figura en la [documentación sobre la configuración manual](https://github.com/mill1000/midea-ac-py#manual-configuration).

Por ejemplo, una PortaSplit fue detectada en el rastreador de incidencias de `Midea AC LAN` como tipo de dispositivo `0xAC`, modelo `00000Q1D` y versión de protocolo 3. Posteriormente, el mismo usuario pudo añadirla a Home Assistant mediante NetHome Plus. El historial concreto está documentado en [Issue #607](https://github.com/wuwentao/midea_ac_lan/issues/607).

La separación es decisiva:

- El servicio en la nube se utiliza para obtener las credenciales locales.
- El control posterior se realiza directamente en la LAN.
- Por tanto, una interrupción del servicio de tokens afecta principalmente a nuevas configuraciones.
- No finaliza automáticamente una conexión local ya configurada.

Esto último también corresponde a la descripción explícita de [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

## De dónde procede la advertencia de desactivación

El texto de advertencia visible hoy se incorporó a la documentación el 19 de mayo de 2025 mediante [Pull Request #578](https://github.com/wuwentao/midea_ac_lan/pull/578).

La justificación puede resumirse así:

- Los tokens locales no tendrían fecha de caducidad.
- Varios proyectos de Home Assistant utilizarían cifrado de aplicaciones reproducido o extraído.
- De ello se derivaría un problema de seguridad.
- Por ello, Midea cerraría gradualmente los servicios de tokens existentes.
- A largo plazo, el control local V1 sería desplazado por una API V2 basada en la nube.

En julio de 2025, la documentación se adaptó de nuevo mediante [Pull Request #639](https://github.com/wuwentao/midea_ac_lan/pull/639). En lugar de la nube de SmartHome, ahora se mencionaba NetHome Plus como fuente de tokens utilizada temporalmente. La advertencia de desactivación propiamente dicha se mantuvo.

Sin embargo, el debate subyacente está formulado con más cautela que la README.

En el [comentario del mantenedor de Midea-AC-LAN](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457) se indica, en esencia, que NetHome Plus podría ser solo una solución temporal y que, según su entendimiento, Midea dispone de un nuevo servicio V2 completamente basado en la nube.

El mantenedor de `midea-msmart` respondió que también sospechaba la existencia de una nueva API V2, pero que, al no disponer de dispositivos Midea propios, solo podía investigarla de forma limitada. Esto figura en el [comentario de respuesta directa](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109).

Con ello, la situación de las fuentes queda más clara:

- La advertencia procede de desarrolladores experimentados de la comunidad.
- Se basa en cambios observados y en su evaluación técnica.
- Uno de los mantenedores califica expresamente la migración a V2 como su interpretación.
- El otro habla de una sospecha.
- Ni el Pull Request ni el debate enlazan un anuncio oficial de Midea sobre la desactivación ni una fecha.

Esto no hace que la advertencia carezca de valor. Pero la convierte en un análisis de riesgo, no en una hoja de ruta confirmada por el fabricante.

## El hallazgo decisivo de junio de 2026

El 15 de junio de 2026 se integró en la biblioteca `midea-local` una corrección que modifica sustancialmente la interpretación anterior.

El punto de partida fue el error:

```json
{
  "code": "3004",
  "msg": "value is illegal."
}
```

Este error se había producido al consultar el token y la clave mediante la nube de SmartHome. El inicio de sesión y la lista de dispositivos seguían funcionando, pero la llamada a `/v1/iot/secure/getToken` era rechazada.

Al principio, esto parecía una interfaz desactivada o inutilizada. Sin embargo, un análisis de la solicitud de la aplicación oficial de SmartHome mostró otra causa: además de `udpid`, la aplicación enviaba el campo `applianceCodes`. La biblioteca de la comunidad no enviaba este campo.

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
- Los valores devueltos funcionaban después para la autenticación local V3.

La investigación completa, los resultados de las pruebas y el diff de código están documentados en [`midea-local` Pull Request #470](https://github.com/midea-lan/midea-local/pull/470). El commit inmutable correspondiente está en [`23312799`](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5).

El código fuente actual sigue utilizando exactamente este endpoint:

```text
/v1/iot/secure/getToken
```

Además, ahora también se envía `applianceCodes`. Esto puede comprobarse directamente en el [`midealocal/cloud.py`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) actual.

La versión actual de `Midea AC LAN` integra `midea-local==6.11.0` y sigue declarándose como integración `local_push`. Ambos datos figuran en el [`manifest.json`](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json) actual.

Por tanto, la afirmación general de que la API de tokens de SmartHome se había cerrado queda refutada, al menos para las cuentas y los dispositivos probados en junio de 2026. Lo correcto sería:

> La consulta de tokens anterior dejó de funcionar tras un cambio en el formato de solicitud esperado. Después de adaptarla al formato utilizado por la aplicación oficial, el mismo endpoint V1 volvió a proporcionar credenciales locales válidas.

Con ello no se excluyen diferencias regionales, cuentas distintas o tipos de dispositivos no compatibles. Pero evidentemente no se trataba de una desactivación global.

## Por qué «V2» se malinterpreta tan fácilmente aquí

En el entorno de Midea se utilizan al menos tres denominaciones de versión independientes entre sí.

| Término | Significado |
| --- | --- |
| Protocolo local V2/V3 | Generación de la comunicación directa entre la integración y el dispositivo |
| Endpoint de aplicación V1/V2 | Número de versión de un endpoint HTTP individual en el backend de las aplicaciones de Midea |
| API V2 de nube a nube | API oficial para socios para empresas externas autorizadas |

### V2 y V3 locales

En el protocolo local de dispositivos, V2 o V3 designa la generación de comunicación del dispositivo. Los dispositivos V3 más recientes necesitan token y clave para la autenticación local. `Midea Smart AC` documenta este requisito en su [guía de configuración](https://github.com/mill1000/midea-ac-py#manual-configuration).

Esta versión de protocolo no tiene nada que ver con la API V2 oficial de nube a nube.

### V1 y V2 en URL de aplicaciones

Incluso dentro de la misma aplicación pueden utilizarse simultáneamente endpoints con números de versión diferentes. Por ello, un `/v2/` en la ruta de la URL no significa que toda la plataforma se haya migrado a una nueva arquitectura.

El código actual de `midea-local` sigue utilizando [`/v1/iot/secure/getToken`](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py) para token y clave. Otras funciones pueden encontrarse igualmente en rutas con versiones diferentes.

### API V2 oficial de nube a nube

Midea documenta efectivamente una [API V2 oficial de nube a nube](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Esta utiliza, entre otros:

- OAuth 2.0
- `client_id` y `client_secret`
- tokens de acceso y tokens de actualización de corta duración
- firmas HMAC-SHA256
- `/v2/open/oauth2/authorize`
- `/v2/open/oauth2/token`
- `/v2/open/device/list/get`
- consultas de estado y comandos de control basados en la nube

Se trata de una interfaz controlada para socios. El `client_secret` necesario es asignado por Midea a un proveedor externo. Un propietario normal de una PortaSplit no lo obtiene simplemente a través de su cuenta MSmartHome. Los requisitos y las reglas de firma se describen en la [documentación oficial de V2](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html).

Además, esta API no surgió por primera vez en 2025. La documentación contiene ejemplos de solicitudes con marcas de tiempo de 2018 y un comentario Java del 18 de abril de 2019. Por tanto, la interfaz V2 para socios ya existía mucho antes de la advertencia en `Midea AC LAN`.

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

Y la conexión local de PortaSplit ya no pasa por una URL en la nube de este tipo, sino directamente por la red doméstica.

Por tanto, equiparar las tres interfaces únicamente por el número de versión «V1» no estaría técnicamente justificado.

## ¿Ya existe una integración de Home Assistant completamente basada en la nube?

Con [`Midea Auto Cloud`](https://github.com/sususweet/midea_auto_cloud) ya existe una integración de la comunidad que controla dispositivos Midea mediante la nube en lugar de directamente por la LAN.

Sin embargo, tampoco esto demuestra que la API V2 oficial para socios ya haya sustituido el control local. El código fuente actual de `Midea Auto Cloud` utiliza, entre otros:

```text
/v1/appliance/transparent/send
/mjl/v1/device/status/lua/get
/mjl/v1/device/lua/control
```

Estos endpoints pueden consultarse en el [`core/cloud.py`](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py) actual.

La integración reproduce así funciones privadas de aplicaciones o de la nube de consumo. No utiliza simplemente la interfaz para socios documentada `/v2/open/...`.

Por tanto, ya existe una alternativa basada en la nube. Pero también conlleva las dependencias habituales de una integración en la nube: acceso a Internet, una cuenta de usuario funcional, servidores de Midea disponibles y endpoints privados que sigan siendo compatibles.

## ¿Qué significa esto concretamente para los propietarios de una PortaSplit?

### Control local ya configurado

Para una PortaSplit ya configurada, la situación es relativamente poco crítica. `Midea Smart AC` guarda el token y la clave localmente tras la configuración y, según su propia [documentación sobre la nube](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage), no necesita una conexión a la nube para el control posterior.

Por tanto, la desactivación de la mera obtención de tokens no finalizaría automáticamente la conexión local existente.

### Nueva configuración o restauración

El riesgo es mayor en caso de:

- una nueva instalación de Home Assistant
- cambiar a otra integración
- una copia de seguridad perdida o dañada
- sustituir el módulo Wi-Fi
- cambios en la asignación del dispositivo
- un nuevo emparejamiento, si con ello cambian las credenciales del dispositivo

En estos casos, la integración debe volver a obtener el token y la clave, o el usuario debe introducirlos manualmente. Que `Midea Smart AC` admite una configuración manual se describe en su [documentación de configuración](https://github.com/mill1000/midea-ac-py#manual-configuration).

No está documentado oficialmente si un restablecimiento de fábrica o un nuevo emparejamiento genera necesariamente nuevas credenciales en cada PortaSplit, por lo que no debe afirmarse de manera general.

### Una desactivación real del control por LAN

Para que una PortaSplit ya configurada deje de aceptar sus credenciales almacenadas localmente, tendría que cambiar además el comportamiento del dispositivo o del módulo Wi-Fi, por ejemplo mediante un nuevo firmware o un procedimiento de autenticación modificado.

La mera desactivación del endpoint en la nube `/v1/iot/secure/getToken` no elimina automáticamente las credenciales que ya existen en el dispositivo y en Home Assistant. Esto se deriva de la separación entre la obtención única desde la nube y el posterior control por LAN documentada por [`Midea Smart AC`](https://github.com/mill1000/midea-ac-py#note-on-cloud-usage).

Un cambio futuro de este tipo en el dispositivo es técnicamente posible. Sin embargo, no he encontrado en la documentación pública de Midea un anuncio concreto ni una fecha de desactivación específica para la PortaSplit.

## Lo que seguiría recomendando

Pese a los hallazgos que relativizan la situación, una copia de seguridad sigue teniendo sentido.

Para los dispositivos V3, `Midea AC LAN` recomienda expresamente guardar fuera de HAOS la configuración JSON generada. La recomendación actual figura directamente en la [README del proyecto](https://github.com/wuwentao/midea_ac_lan#1-important-notice).

Se aplica lo siguiente:

- Tratar el token y la clave como contraseñas.
- No subir el archivo JSON a un repositorio Git público.
- No publicar registros de depuración sin censurar.
- Cifrar la copia de seguridad.
- Crear además una copia de seguridad completa de Home Assistant.
- Comprobar el funcionamiento actual antes de actualizaciones de firmware e integración.
- Volver a probar el control local después de las actualizaciones.

Una copia de seguridad es una protección razonable frente a cambios en la nube, problemas de integración y errores propios. Pero no indica que una desactivación sea inminente. Cómo configurar correctamente una PortaSplit y protegerla en la red doméstica se explica en la [parte práctica sobre la configuración](/blog/midea-portasplit-home-assistant-einrichten).

## Evaluación basada en las pruebas disponibles

La advertencia de `Midea AC LAN` debe tomarse en serio, pero debe contextualizarse correctamente.

Documenta un riesgo plausible a largo plazo: Midea podría considerar los tokens locales sin caducidad como un problema de seguridad, restringir aún más la obtención de tales tokens o vincular más estrechamente los dispositivos futuros a la nube.

En cambio, no está demostrado un cierre oficialmente anunciado y programado del control local de PortaSplit.

El estado técnico actual incluso muestra lo contrario de una desactivación ya consumada: en junio de 2026, el endpoint de tokens V1 todavía utilizado proporcionaba credenciales válidas después de que la solicitud se adaptara al formato de la aplicación oficial de SmartHome. La corrección correspondiente forma hoy parte de la biblioteca utilizada por `Midea AC LAN`.

La API V2 oficial de Midea de nube a nube también existe. Sin embargo, es una interfaz para socios más antigua y de acceso restringido, y no es automáticamente la sucesora del protocolo local de PortaSplit.

Por tanto, la conclusión sobria es:

> Crear una copia de seguridad, vigilar las integraciones y tener presentes las dependencias de la nube, pero no descartar prematuramente el control local de PortaSplit basándose en una suposición de desactivación no confirmada.

## Fuentes

1.  [Midea AC LAN: README actual y advertencia de desactivación](https://github.com/wuwentao/midea_ac_lan#1-important-notice): texto de la advertencia, recomendación de copia de seguridad y distinción entre dispositivos V2 más antiguos y V3 más recientes.

2.  [Midea AC LAN PR #578 del 19 de mayo de 2025](https://github.com/wuwentao/midea_ac_lan/pull/578): introducción de la advertencia sobre el cierre gradual de los servicios de tokens y la supuesta migración a una API V2 basada en la nube.

3.  [Midea AC LAN PR #639](https://github.com/wuwentao/midea_ac_lan/pull/639): cambio de la fuente de tokens documentada a NetHome Plus.

4.  [midea-msmart Issue #201](https://github.com/mill1000/midea-msmart/issues/201): debate sobre la consulta errónea de tokens de SmartHome y el uso temporal de NetHome Plus.

5.  [Comentario del mantenedor de Midea-AC-LAN sobre la supuesta migración a V2](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2746782457): identifica expresamente la afirmación sobre la nueva nube V2 como su propia interpretación.

6.  [Respuesta del mantenedor de midea-msmart](https://github.com/mill1000/midea-msmart/issues/201#issuecomment-2751782109): describe la existencia de una nueva API V2 como una sospecha y señala las posibilidades limitadas de ingeniería inversa.

7.  [midea-local PR #470 del 15 de junio de 2026](https://github.com/midea-lan/midea-local/pull/470): análisis del error 3004, captura de la solicitud de la aplicación oficial, incorporación de `applianceCodes` y prueba satisfactoria con cuatro equipos de aire acondicionado V3.

8.  [Commit inmutable de la corrección SmartHome-getToken](https://github.com/midea-lan/midea-local/commit/23312799bbe80576f869c582f505dcfabf31aed5): diff de código exacto de la corrección integrada.

9.  [Código actual de la nube de midea-local](https://github.com/midea-lan/midea-local/blob/main/midealocal/cloud.py): endpoint aún utilizado `/v1/iot/secure/getToken` y campo de solicitud actual `applianceCodes`.

10.  [Manifiesto actual de Midea AC LAN](https://github.com/wuwentao/midea_ac_lan/blob/main/custom_components/midea_ac_lan/manifest.json): versión utilizada de `midea-local` y clasificación como integración push local.

11.  [Midea Smart AC](https://github.com/mill1000/midea-ac-py): documentación del control local, de la obtención única desde la nube para dispositivos V3 y de la configuración manual con token y clave.

12.  [Midea AC LAN Issue #607 sobre la PortaSplit](https://github.com/wuwentao/midea_ac_lan/issues/607): ejemplo concreto de PortaSplit con tipo de dispositivo `0xAC`, modelo `00000Q1D`, versión de protocolo 3 y configuración satisfactoria mediante NetHome Plus.

13.  [API V2 oficial de Midea de nube a nube](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-v2-api.html): OAuth2, ID de cliente, secreto de cliente, tokens de acceso y actualización, procedimiento de firma y endpoints `/v2/open/...`.

14.  [API V1 oficial de Midea de nube a nube](https://mis-cdn.smartmidea.net/docs/control-midea-cloud-devices/cloud-2-cloud-api.html): aviso oficial de que la antigua interfaz para socios `/v1/open/...` ya no se recomienda y podría desactivarse en el futuro.

15.  [Midea Auto Cloud](https://github.com/sususweet/midea_auto_cloud) y [código actual de la nube](https://github.com/sususweet/midea_auto_cloud/blob/master/custom_components/midea_auto_cloud/core/cloud.py): integración de la comunidad para control completo en la nube y los endpoints privados V1 de aplicaciones realmente utilizados.
