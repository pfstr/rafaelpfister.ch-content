---
title: "F5 BIG-IP como proxy saliente para envíos masivos de correo: persistencia, SNAT, tiempos de espera y resolución DNS"
navTitle: "F5 envíos masivos"
description: "Un envío masivo de 1000 correos por minuto pasa por una BIG-IP como proxy saliente hacia el relay del proveedor. El artículo explica por qué las sesiones persistentes no aportan nada en este caso, cómo resolver correctamente el nombre de host del proveedor mediante un nodo FQDN y qué ajustes de SNAT, tiempos de espera y límites de conexión determinan realmente el rendimiento."
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
aiPrompt: |
  Du bist mein Netzwerk- und Mailflow-Assistent. Wir versenden Massenmails über eine F5 BIG-IP als ausgehenden Proxy zu einem Provider-Relay. Hilf mir, die BIG-IP-Konfiguration nach diesem Artikel zu prüfen: 1. Frage mich nach Versandrate, Anzahl paralleler Verbindungen und Nachrichten pro Verbindung. 2. Frage nach Virtual-Server-Typ, Persistenzprofil, Idle-Timeout und SNAT-Konfiguration. 3. Prüfe, ob der Provider-Hostname als FQDN-Node mit Autopopulate hinterlegt ist und ob DNS-Server auf der BIG-IP konfiguriert sind. 4. Nenne mir konkrete Abweichungen von den Empfehlungen aus dem Artikel und begründe jede Änderung.
translationOf: f5-big-ip-outbound-smtp-massenversand
url: https://rafaelpfister.ch/es/blog/f5-big-ip-como-proxy-saliente-para-envios-masivos-de-correo-persistencia-snat-tiempos-de-espera
translationSourceHash: 218c4d189dd18000d6db2ead4b2106f8be858169c9d7b234e4f9320ac802fd46
translationModel: gpt-5.6-terra
translatedAt: 2026-08-27T14:29:28.536Z
translationReview: automatic
---

Una ejecución de facturación o el envío de un boletín con unas 1000 correos por minuto sale de la red corporativa, con una F5 BIG-IP actuando como proxy saliente hacia el punto de entrega del proveedor. La BIG-IP no distribuye hacia varios destinos, sino que reenvía el tráfico. Precisamente esta configuración determina qué ajustes tienen sentido y qué supuestas optimizaciones no sirven de nada.

## La arquitectura en una frase

Los sistemas de envío se conectan como smarthost a una dirección interna de servidor virtual en la BIG-IP; la BIG-IP traduce las direcciones de origen mediante SNAT a una IP pública fija y reenvía cada conexión al nombre de host del proveedor. No hay balanceo de carga propiamente dicho en la BIG-IP, porque el pool solo tiene un miembro. Parece una configuración trivial, pero las decisiones de detalle (persistencia, tiempos de espera, tipo de SNAT, resolución DNS) determinan si el envío funciona de forma estable o presenta interrupciones inexplicables bajo carga.

## ¿Son mejores las sesiones persistentes? No, por dos motivos

La cuestión de la persistencia de sesión procede del mundo HTTP, donde un usuario con carrito de compra o sesión de inicio de sesión debe llegar siempre al mismo backend. Trasladado a SMTP, el concepto no tiene sentido.

En primer lugar, SMTP concluye sin estado por conexión: cada conexión procesa una o varias transacciones completas (MAIL FROM, RCPT TO, DATA) y termina con QUIT. No existe ningún estado que deba permanecer en el mismo sistema de destino entre conexiones. Qué sistema del lado del proveedor acepta la siguiente conexión es irrelevante para la entrega.

En segundo lugar, en esta BIG-IP simplemente no hay nada que persistir: el pool contiene exactamente un miembro, la única dirección IP del proveedor. Un perfil de persistencia solo consumiría memoria para una tabla de persistencia y requeriría una búsqueda en cada conexión que siempre devolvería el mismo resultado. Por tanto, el ajuste correcto es: Default Persistence Profile en None. Incluso si el proveedor publicara más adelante varias direcciones IP tras el nombre de host, la persistencia sería contraproducente, ya que impediría la distribución entre esas direcciones y cargaría unilateralmente destinos individuales.

Lo decisivo para el rendimiento en los envíos masivos es el perfil de conexión del emisor: pocas conexiones de larga duración con muchos mensajes por conexión, en lugar de una conexión nueva por correo; más sobre ello abajo.

## Servidor virtual: FastL4 en lugar de Full Proxy

Para el simple reenvío de SMTP, un servidor virtual Performance (capa 4) con perfil FastL4 es la elección correcta. La BIG-IP procesa entonces la conexión en gran medida en hardware o en la ruta acelerada, sin terminar por completo la conexión TCP. Un servidor virtual estándar en modo Full Proxy solo aporta valor si realmente desea intervenir en el flujo de datos en la BIG-IP, por ejemplo con un perfil de seguridad SMTP o iRules a nivel de protocolo. Para un proxy saliente hacia su propio proveedor contratado, esto es innecesario y solo crea fuentes adicionales de errores.

Importante en ambos casos: no active ningún perfil que escriba en la conexión. Los sistemas de envío negocian STARTTLS directamente con el relay del proveedor; cualquier instancia que modifique o filtre bytes pone en riesgo el establecimiento de TLS.

## Resolución DNS: el nombre de host del proveedor debe estar como nodo FQDN en el pool

El proveedor ha proporcionado un nombre de host, no una dirección IP. El reflejo obvio de resolver la IP una vez e introducirla estáticamente como nodo es la peor variante: si el proveedor cambia la dirección (mantenimiento, migración, caso de DR), el envío se detiene hasta que alguien adapte la configuración de la BIG-IP. Precisamente para eso existen los nodos FQDN.

Un nodo FQDN almacena el nombre de host en lugar de la dirección. La BIG-IP resuelve el nombre por sí misma, crea para cada dirección devuelta un denominado nodo efímero y los actualiza automáticamente cuando cambia la respuesta DNS. De forma predeterminada, consulta el nombre de nuevo al expirar el TTL de DNS; alternativamente, puede establecerse un intervalo de consulta fijo. Con Autopopulate activado, el pool adopta automáticamente también varios registros A como miembros: si el proveedor amplía posteriormente su entrega a varias direcciones, la BIG-IP lo sigue sin cambios de configuración.

Hay dos requisitos que suelen olvidarse. En primer lugar, la BIG-IP necesita servidores DNS funcionales en la configuración del sistema (System, Configuration, Device, DNS); los nodos FQDN utilizan los resolvedores del sistema, no una caché DNS de un perfil de listener. En segundo lugar, estos resolvedores deben ser realmente accesibles desde el contexto de gestión o TMM; de lo contrario, el nodo permanecerá en estado unresolved y el pool estará vacío.

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

A continuación, los sistemas de envío configuran 10.0.5.10 como smarthost. El proveedor determina si se utiliza el puerto 25 o 587; la configuración de la BIG-IP es idéntica en ambos casos, solo cambia el puerto.

## SNAT: dirección fija en lugar de Automap

En el tráfico de correo saliente, la dirección de origen debe estar bajo control. SNAT Automap toma la Floating Self-IP de la VLAN saliente, y esta puede cambiar inadvertidamente debido a modificaciones de red o cambios de failover. Sin embargo, los proveedores suelen vincular la entrega a una lista de IP permitidas e, incluso sin una lista formal, la reputación depende de la dirección de origen. Un pool SNAT dedicado con una dirección asignada de forma fija convierte la IP de origen en un objeto de configuración documentado y estable.

En cuanto a capacidad: una única dirección SNAT ofrece frente a un único destino (una IP, un puerto) unas 64'000 traducciones simultáneas, ya que cada conexión recibe su propio puerto de origen efímero. Con el perfil de carga descrito aquí, de unas pocas docenas de conexiones simultáneas, esto es suficiente por varios órdenes de magnitud. El agotamiento de puertos solo se convierte en un problema cuando un emisor mal configurado abre una nueva conexión por correo y no la cierra correctamente; entonces las traducciones se acumulan en un estado similar a TIME-WAIT. Este comportamiento debe corregirse en el emisor, no con una segunda dirección SNAT.

## Tiempos de espera: la causa más frecuente de interrupciones de conexión bajo carga

Un emisor masivo mantiene las conexiones abiertas y envía mensaje tras mensaje. Entre dos mensajes pueden surgir pausas: el emisor genera el siguiente bloque o el relay retrasa la aceptación (tarpitting, restos de greylisting, colas internas). El tiempo de espera de inactividad del perfil FastL4 está establecido de forma predeterminada en 300 segundos. Si una pausa supera ese valor, la BIG-IP elimina la conexión y el emisor escribe en una conexión que ya no existe.

Dos ajustes mitigan este problema. En primer lugar, establezca el tiempo de espera de inactividad en un valor superior a las pausas realistas; para envíos masivos, 600 segundos es un valor inicial razonable. No debería elevarse arbitrariamente, porque de lo contrario se acumulan conexiones huérfanas en la tabla de conexiones. En segundo lugar, mantenga activado Reset on Timeout en el perfil: la BIG-IP confirma entonces la eliminación con un TCP reset, y el MTA emisor detecta de inmediato que la conexión ha desaparecido, en lugar de llegar a un timeout y volver a programar el mensaje solo después de varios minutos.

No tiene influencia sobre los tiempos de espera del otro extremo, pero deben tenerse en cuenta: si el relay del proveedor cierra conexiones tras 120 segundos de inactividad, un tiempo de espera generoso en la BIG-IP no sirve de nada. El valor determinante es el menor tiempo de espera de toda la ruta; en caso de duda, consulte al proveedor y use ese valor como base de planificación.

## Estrategia de conexión: pocas conexiones, muchos mensajes

Sin especificaciones de entrega del proveedor, merece la pena hacer un cálculo breve. 1000 correos por minuto son unos 17 por segundo. Una transacción SMTP sobre una conexión ya establecida dura claramente menos de medio segundo con una latencia normal. Con 10 a 20 conexiones paralelas y, por ejemplo, 100 mensajes por conexión antes de que el emisor las renueve, la tasa objetivo se alcanza cómodamente. Por lo general, el proveedor dispone de bastante más capacidad de conexión, pero esta se comparte con todos los demás clientes. Por ello, pocas conexiones de larga duración con muchas transacciones no solo son eficientes (se evita establecer TCP y TLS por mensaje), sino también la forma más respetuosa de utilizar infraestructura ajena.

Los ajustes para ello están en el sistema de envío, no en la BIG-IP: máximo de mensajes por conexión, máximo de conexiones paralelas al smarthost y reutilización de conexiones establecidas. En la BIG-IP, todo ello puede protegerse con un Connection Limit en el miembro del pool, por ejemplo 200 conexiones simultáneas: en funcionamiento normal nunca se alcanza ese valor, pero un emisor mal configurado que de repente abre una conexión por correo no inundará sin límites el relay del proveedor. El límite es una red de seguridad, no un instrumento de control.

La medición muestra si el perfil de conexión configurado se cumple en la práctica: las conexiones por minuto y los mensajes por conexión pueden evaluarse a partir del Message Tracking o de los logs del conector, tal como se describe en el artículo [Determinar el perfil de carga de un servidor de correo](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln). Para una prueba de carga con un patrón masivo realista (pocas sesiones, muchos mensajes por sesión), smtp-source del paquete Postfix es más adecuado que las herramientas de carga orientadas a HTTP, porque genera precisamente este perfil de conexión.

## Monitorización: no sobrecargue al proveedor con comprobaciones de estado

Un monitor en el miembro del pool es útil para que la BIG-IP detecte un fallo en el lado del proveedor y lo notifique correctamente. Se aplica lo siguiente: cada comprobación de estado es una conexión real con el proveedor y cuenta frente a los mismos límites que el tráfico de uso. Un monitor TCP sencillo con un intervalo moderado (30 segundos o más) es más que suficiente. Un monitor SMTP completo que compruebe hasta el banner o EHLO apenas aporta información adicional, pero genera entradas de registro en el lado del proveedor y, en el peor de los casos, consultas sobre por qué llega una conexión sin correo cada 5 segundos.

## Lista de comprobación

| Ajuste | Recomendación |
|---|---|
| Perfil de persistencia | None; las sesiones persistentes no aportan nada con SMTP y menos aún en un pool de un solo miembro |
| Tipo de servidor virtual | Performance (capa 4) con perfil FastL4, sin intervención en el flujo de datos |
| Nodo de destino | Nodo FQDN con Autopopulate en lugar de IP estática; servidores DNS configurados en la BIG-IP |
| SNAT | Pool SNAT dedicado con dirección fija conocida por el proveedor; sin Automap |
| Tiempo de espera de inactividad | Por encima de las pausas reales de envío, valor inicial de 600 s; Reset on Timeout activo |
| Connection Limit | Como red de seguridad en el miembro del pool, p. ej., 200 |
| Monitor | TCP, intervalo de 30 s o más; sin monitor SMTP agresivo |
| Configuración del emisor | Pocas conexiones paralelas, muchos mensajes por conexión; reutilización activa |

La respuesta breve a la pregunta inicial es, por tanto: no, las sesiones persistentes no son mejores; en esta configuración son ineficaces o incluso perjudiciales. La calidad de la solución depende de la resolución DNS del nombre de host del proveedor, de una dirección SNAT estable, de tiempos de espera adecuados para el perfil de carga y de que los sistemas de envío entreguen sus 1000 correos por minuto mediante unas pocas conexiones establecidas en lugar de mil individuales.

## Fuentes

1.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321): la sección 4.5.4 y el modelo de transacción muestran que varias transacciones de correo a través de una conexión son el caso normal previsto.

2.  [K7820: Overview of SNAT features](https://my.f5.com/manage/s/article/K7820): artículo básico de F5 sobre SNAT, pools SNAT y la traducción de puertos por destino.

3.  [Referencia de tmsh: ltm node](https://clouddocs.f5.com/cli/tmsh-reference/latest/modules/ltm/ltm_node.html): documenta las opciones FQDN (name, autopopulate, interval) para nodos y, por tanto, para miembros del pool.

4.  [smtp-source(1), Postfix](https://www.postfix.org/smtp-source.1.html): generador de carga que reproduce el perfil de conexión de un emisor masivo (pocas sesiones, muchos mensajes).

5.  [Determinar el perfil de carga de un servidor de correo](https://rafaelpfister.ch/blog/mailserver-lastprofil-ermitteln): guía propia sobre cómo evaluar las conexiones por minuto y los mensajes por conexión a partir de Message Tracking.
