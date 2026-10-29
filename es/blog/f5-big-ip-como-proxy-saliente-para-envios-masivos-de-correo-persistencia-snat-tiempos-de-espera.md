---
title: "F5 BIG-IP como proxy saliente para envíos masivos de correo: persistencia, SNAT, tiempos de espera y resolución DNS"
navTitle: "F5 envíos masivos"
description: "Un envío masivo de 1000 correos por minuto pasa por una BIG-IP como proxy saliente hacia el relay del proveedor. El artículo explica por qué las sesiones sticky no aportan nada en este caso, cómo resolver correctamente el nombre de host del proveedor mediante un nodo FQDN y qué ajustes de SNAT, tiempos de espera y límites de conexión determinan realmente el rendimiento."
date: "2026-08-26"
kategorie: "Balanceador de carga"
timeToRead: "9 min de lectura"
themen:
  - loadbalancer
  - smtp-mailflow
produkte:
  - "loadbalancer"
protokolle:
  - "smtp"
  - "tcp"
  - "dns"
hauptthema: "loadbalancer"
related:
  - massenmailing-provider-wechsel-checkliste
  - mailserver-lastprofil-ermitteln
slug: "f5-big-ip-como-proxy-saliente-para-envios-masivos-de-correo-persistencia-snat-tiempos-de-espera"
featured: true
translationId: "article-ee5e63e82ffd2604"
translationOf: f5-big-ip-outbound-smtp-massenversand
translationSourceHash: 5c62d1467d3a000a8c4a3255d6f925e294d76d1b65054e91b863a5166e1a22f8
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:23:39.980Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/f5-big-ip-como-proxy-saliente-para-envios-masivos-de-correo-persistencia-snat-tiempos-de-espera
---

Una ejecución de facturación o el envío de un boletín con unas 1000 correos por minuto sale de la red corporativa, con una F5 BIG-IP como proxy saliente hacia el punto de entrega del proveedor. La BIG-IP no distribuye hacia varios destinos, sino que retransmite. Precisamente esta configuración determina qué ajustes tienen sentido y qué supuestas optimizaciones no producen ningún efecto.

## La arquitectura en una frase

Los sistemas de envío se comunican con una dirección interna de servidor virtual de la BIG-IP como smarthost; la BIG-IP traduce las direcciones de origen mediante SNAT a una IP pública fija y reenvía cada conexión al nombre de host del proveedor. No hay balanceo de carga propiamente dicho en la BIG-IP, porque el pool solo tiene un miembro. Parece una configuración trivial, pero las decisiones de detalle (persistencia, tiempos de espera, tipo de SNAT, resolución DNS) determinan si el envío funciona de forma estable o muestra interrupciones inexplicables bajo carga.

## ¿Son mejores las sesiones sticky? No, por dos motivos

La cuestión de la persistencia de sesión procede del mundo HTTP, donde un usuario con carrito de compra o sesión iniciada siempre debe llegar al mismo backend. Aplicado a SMTP, el concepto no tiene sentido.

En primer lugar, SMTP se completa sin estado por conexión: cada conexión procesa una o varias transacciones completas (MAIL FROM, RCPT TO, DATA) y termina con QUIT. No existe ningún estado que deba permanecer en el mismo sistema de destino entre conexiones. Qué sistema del lado del proveedor acepta la siguiente conexión es irrelevante para la entrega.

En segundo lugar, en esta BIG-IP no hay literalmente nada que persistir: el pool contiene exactamente un miembro, la única dirección IP del proveedor. Un perfil de persistencia solo consumiría memoria para una tabla de persistencia e implicaría una búsqueda en cada conexión que siempre devuelve el mismo resultado. Por ello, el ajuste correcto es: Default Persistence Profile en None. Incluso si más adelante el proveedor publicara varias direcciones IP detrás del nombre de host, la persistencia sería contraproducente, porque impediría la distribución entre esas direcciones y cargaría de forma desigual destinos individuales.

Lo decisivo para el rendimiento en envíos masivos es el perfil de conexión del remitente: pocas conexiones duraderas con muchos mensajes por conexión, en lugar de una conexión nueva por cada correo; más sobre ello abajo.

## Virtual Server: FastL4 en lugar de Full Proxy

Para el simple reenvío de SMTP, un Virtual Server de rendimiento (capa 4) con perfil FastL4 es la opción adecuada. La BIG-IP procesa entonces la conexión en gran medida mediante hardware o en la ruta acelerada, sin terminar completamente la conexión TCP. Un Virtual Server estándar en modo Full Proxy solo aporta valor si realmente desea intervenir en el flujo de datos en la BIG-IP, por ejemplo con un perfil de seguridad SMTP o iRules a nivel de protocolo. Para un proxy saliente hacia el proveedor contratado, esto es innecesario y solo crea fuentes de error adicionales.

Importante en ambos casos: no active ningún perfil que escriba dentro de la conexión. Los sistemas de envío negocian STARTTLS directamente con el relay del proveedor; cualquier instancia que modifique o filtre bytes pone en riesgo el establecimiento de TLS.

## Resolución DNS: el nombre de host del proveedor debe estar como nodo FQDN en el pool

El proveedor ha proporcionado un nombre de host, no una dirección IP. El reflejo obvio de resolver la IP una vez e introducirla estáticamente como nodo es la peor variante: si el proveedor cambia la dirección (mantenimiento, migración, caso de DR), el envío se detiene hasta que alguien adapte la configuración de la BIG-IP. Para eso existen los nodos FQDN.

Un nodo FQDN guarda el nombre de host en lugar de la dirección. La BIG-IP resuelve el nombre por sí misma, crea un llamado nodo efímero para cada dirección devuelta y los actualiza automáticamente cuando cambia la respuesta DNS. Por defecto consulta el nombre cada 3600 segundos (una hora); con `interval ttl` se guía en cambio por el TTL de la respuesta DNS; alternativamente, puede definirse otro intervalo fijo de consulta. Con Autopopulate activado, el pool también incorpora automáticamente varios registros A como miembros: si el proveedor amplía posteriormente su entrega a varias direcciones, la BIG-IP lo sigue sin cambios de configuración.

A menudo se olvidan dos requisitos. Primero, la BIG-IP necesita servidores DNS funcionales en la configuración del sistema (System, Configuration, Device, DNS); los nodos FQDN usan los resolvedores del sistema, no una caché DNS de un perfil de listener. Segundo, estos resolvedores deben ser realmente accesibles desde el contexto de gestión o TMM; de lo contrario, el nodo permanece en estado unresolved y el pool queda vacío.

La configuración en tmsh tiene este aspecto (las direcciones y los nombres son ejemplos):

```bash
tmsh create ltm node relay-provider fqdn { \
  name mail-relay.provider.example autopopulate enabled }

tmsh create ltm pool pool_provider_smtp \
  members add { relay-provider:25 } monitor tcp

tmsh create ltm snatpool snat_mailout \
  members add { 198.51.100.10 }

tmsh create ltm virtual vs_mailout_smtp \
  destination 10.0.5.10:25 ip-protocol tcp \
  profiles add { fastL4 } pool pool_provider_smtp \
  source-address-translation { type snat pool snat_mailout }
```

Los sistemas de envío configuran después 10.0.5.10 como smarthost. Que utilice el puerto 25 o 587 lo determina el proveedor; la configuración de la BIG-IP es idéntica en ambos casos, solo cambia el puerto.

## SNAT: dirección fija en lugar de Automap

Para el tráfico de correo saliente, la dirección de origen debe estar bajo control. SNAT Automap toma la Floating Self-IP de la VLAN saliente, y esta puede cambiar inadvertidamente durante cambios de red o reconfiguraciones de failover. Sin embargo, los proveedores suelen vincular la entrega a una lista de IP permitidas e, incluso sin una lista formal, la reputación depende de la dirección de origen. Un pool SNAT dedicado con una dirección asignada de forma fija convierte la IP de origen en un objeto de configuración documentado y estable.

En cuanto a capacidad: una única dirección SNAT ofrece frente a un único destino (una IP, un puerto) unas 64'000 traducciones simultáneas, porque cada conexión recibe su propio puerto de origen efímero. Con el perfil de carga descrito aquí, de unas pocas docenas de conexiones simultáneas, esto es suficiente por varios órdenes de magnitud. El agotamiento de puertos solo se convierte en un problema si un remitente mal configurado abre una conexión nueva por cada correo y no la cierra correctamente; entonces las traducciones se acumulan en un estado similar a TIME-WAIT. Ese comportamiento debe corregirse en el remitente, no con una segunda dirección SNAT.

## Tiempos de espera: la causa más frecuente de interrupciones de conexión bajo carga

Un remitente masivo mantiene las conexiones abiertas y envía mensaje tras mensaje. Entre dos mensajes pueden producirse pausas: el remitente genera el siguiente bloque o el relay retrasa la aceptación (tarpitting, restos de greylisting, colas internas). El tiempo de espera de inactividad del perfil FastL4 está configurado por defecto en 300 segundos. Si una pausa lo supera, la BIG-IP elimina la conexión y el remitente escribe en una conexión que ya no existe.

Dos ajustes mitigan esto. Primero, establecer el tiempo de espera de inactividad en un valor superior a las pausas realistas; para envíos masivos, 600 segundos es un valor inicial razonable. El valor no debe ser arbitrariamente alto, pues de lo contrario se acumulan conexiones huérfanas en la tabla de conexiones. Segundo, mantener activado Reset on Timeout en el perfil: entonces la BIG-IP confirma la eliminación con un reset TCP, y el MTA remitente reconoce de inmediato que la conexión se ha perdido, en lugar de esperar a un timeout y reprogramar el mensaje solo al cabo de minutos.

No tiene influencia sobre los tiempos de espera del otro extremo, pero deben tenerse en cuenta: si el relay del proveedor cierra conexiones tras 120 segundos de inactividad, de nada sirve un tiempo de espera generoso en la BIG-IP. El valor determinante es el menor tiempo de espera de toda la ruta; en caso de duda, consulte al proveedor y tome ese valor como base de planificación.

## Estrategia de conexión: pocas conexiones, muchos mensajes

Sin requisitos de entrega por parte del proveedor, merece la pena hacer un cálculo breve. 1000 correos por minuto son unos 17 por segundo. Una transacción SMTP sobre una conexión ya establecida tarda claramente menos de medio segundo con una latencia normal. Con 10 a 20 conexiones paralelas y, por ejemplo, 100 mensajes por conexión antes de que el remitente las renueve, se alcanza cómodamente la tasa objetivo. Por lo general, el proveedor dispone de una capacidad de conexiones considerablemente mayor, pero la comparte con todos los demás clientes. Por ello, pocas conexiones duraderas con muchas transacciones no solo son eficientes (se elimina el establecimiento TCP y TLS por mensaje), sino también la forma más respetuosa de usar infraestructura ajena.

Los ajustes para ello están en el sistema de envío, no en la BIG-IP: máximo de mensajes por conexión, máximo de conexiones paralelas al smarthost y reutilización de conexiones existentes. En la BIG-IP, todo ello puede protegerse con un Connection Limit en el miembro del pool, por ejemplo 200 conexiones simultáneas: en funcionamiento normal nunca se alcanza este valor, pero un remitente mal configurado que de repente abre una conexión por correo no inundará sin control el relay del proveedor. El límite es una red de seguridad, no un instrumento de control.

La medición muestra si el perfil de conexión configurado se cumple en la práctica: las conexiones por minuto y los mensajes por conexión pueden evaluarse a partir del Message Tracking o de los logs del conector, como se describe en el artículo [Determinar el perfil de carga de un servidor de correo](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln). Para una prueba de carga con un patrón de carga masiva realista (pocas sesiones, muchos mensajes por sesión), smtp-source del paquete Postfix es más adecuado que las herramientas de carga orientadas a HTTP, porque genera exactamente este perfil de conexión.

## Monitorización: no cargue al proveedor con health checks

Un monitor en el miembro del pool es útil para que la BIG-IP detecte una caída del lado del proveedor y la notifique correctamente. Se aplica lo siguiente: cada health check es una conexión real con el proveedor y cuenta contra los mismos límites que el tráfico útil. Un monitor TCP simple con un intervalo moderado (30 segundos o más) es más que suficiente. Un monitor SMTP completo, que compruebe hasta el banner o EHLO, apenas aporta información adicional, pero genera entradas de log en el lado del proveedor y, en el peor de los casos, consultas sobre por qué llega cada 5 segundos una conexión sin correo.

## Lista de verificación

| Ajuste | Recomendación |
|---|---|
| Perfil de persistencia | None; las sesiones sticky no aportan nada con SMTP y menos aún en un pool de un solo miembro |
| Tipo de Virtual Server | Performance (Layer 4) con perfil FastL4, sin intervención en el flujo de datos |
| Nodo de destino | Nodo FQDN con Autopopulate en lugar de IP estática; servidores DNS configurados en la BIG-IP |
| SNAT | pool SNAT dedicado con dirección fija conocida por el proveedor; sin Automap |
| Tiempo de espera de inactividad | superior a las pausas reales de envío, valor inicial 600 s; Reset on Timeout activo |
| Connection Limit | como red de seguridad en el miembro del pool, p. ej., 200 |
| Monitor | TCP, intervalo de 30 s o más; sin monitor SMTP agresivo |
| Configuración del remitente | pocas conexiones paralelas, muchos mensajes por conexión; reutilización activa |

Por tanto, la respuesta breve a la pregunta inicial es: no, las sesiones sticky no son mejores; en esta configuración son ineficaces e incluso perjudiciales. La calidad de la solución depende de la resolución DNS del nombre de host del proveedor, de una dirección SNAT estable, de tiempos de espera adaptados al perfil de carga y de que los sistemas de envío entreguen sus 1000 correos por minuto a través de pocas conexiones ya establecidas en lugar de mil conexiones individuales.

## Historial de cambios

- **25 de octubre de 2026:** Corregido: el intervalo predeterminado para la resolución de nombres de nodos FQDN es de 3600 segundos, no el TTL de DNS; la BIG-IP solo se guía por el TTL con el ajuste interval ttl.

## Fuentes

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): la sección 4.5.4 y el modelo de transacción muestran que varias transacciones de correo a través de una conexión son el caso normal previsto.

2.  [K7820: Overview of SNAT features](https://my.f5.com/manage/s/article/K7820): artículo básico de F5 sobre SNAT, pools SNAT y la traducción de puertos por destino.

3.  [Referencia de tmsh: ltm node](https://clouddocs.f5.com/cli/tmsh-reference/latest/modules/ltm/ltm_node.html): documenta las opciones FQDN (name, autopopulate, interval) para nodos y, por tanto, para miembros de pool, incluido el intervalo predeterminado de 3600 segundos y el valor `ttl`.

4.  [smtp-source(1), Postfix](https://www.postfix.org/smtp-source.1.html): generador de carga que reproduce el perfil de conexión de remitente masivo (pocas sesiones, muchos mensajes).

5.  [Determinar el perfil de carga de un servidor de correo](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln): guía propia sobre cómo evaluar conexiones por minuto y mensajes por conexión a partir del Message Tracking.

6.  [F5 Bug ID 746657: tmsh help for FQDN node or pool member shows incorrect default for fqdn interval](https://cdn.f5.com/product/bugtracker/ID746657.html): confirma 3600 segundos como valor predeterminado real del intervalo de consulta FQDN; la indicación anterior de ayuda TTL era incorrecta.
