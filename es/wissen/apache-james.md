---
title: "Apache James: servidor de correo modular y plataforma de Mailets"
blatt: "apache-james"
description: "Apache James en su contexto técnico: protocolos y funciones de correo, arquitectura basada en componentes, cola y canalización de Mailets, modelo de buzón y almacenamiento, variantes operativas desde PostgreSQL hasta Cassandra y evolución desde el proyecto Java Apache hasta una plataforma de correo para la JVM."
fakten:
  - label: Nombre completo
    wert: Java Apache Mail Enterprise Server
    href: https://james.apache.org/
  - label: Categoría
    wert: MTA, MDA, servidor de buzones y plataforma de aplicaciones de correo
    href: https://james.apache.org/documentation.html
  - label: Proyecto
    wert: Apache Software Foundation
    href: https://projects.apache.org/committee.html?james
  - label: Tiempo de ejecución
    wert: JVM · Java 21 a partir de la versión 3.9
    href: https://james.apache.org/james/update/2025/09/25/james-3.9.0.html
  - label: Lenguajes
    wert: principalmente Java, módulos individuales en Scala
    href: https://github.com/apache/james-project
  - label: Protocolos
    wert: SMTP, LMTP, IMAP, POP3, ManageSieve, JMAP
    href: https://james.apache.org/server/feature-protocols.html
  - label: Estilo arquitectónico
    wert: modular, basado en componentes, Inversion of Control, orientado a eventos
    href: https://james.apache.org/
  - label: Backends
    wert: PostgreSQL/JPA o Cassandra · OpenSearch · RabbitMQ · S3
    href: https://james.apache.org/download.cgi
  - label: Configuración
    wert: conf/*.xml y *.properties · variables de entorno
    href: https://james.apache.org/server/config.html
  - label: Empaquetado
    wert: distribuciones ZIP e imágenes oficiales de Docker
    href: https://james.apache.org/download.cgi
  - label: Administración
    wert: WebAdmin REST API, CLI y métricas
    href: https://james.apache.org/server/manage-webadmin.html
  - label: Monitorización
    wert: Health Checks, Prometheus, JMX, registros y Grafana
    href: https://james.apache.org/server/metrics.html
  - label: Licencia
    wert: Apache License 2.0
    href: https://www.apache.org/licenses/LICENSE-2.0
werbung:
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: adbfb005b83b16086ba55e53dd469f3aff1e5642364da5ab8b2da5d265a1ce51
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:24:38.668Z
translationReview: required
---

# Apache James: servidor de correo modular y plataforma de Mailets

Apache James es un servidor de correo de código abierto y, al mismo tiempo, un conjunto de herramientas para aplicaciones cuya lógica de negocio se basa en el correo electrónico. El nombre significa **Java Apache Mail Enterprise Server**. James puede aceptar y reenviar mensajes mediante SMTP, gestionar buzones locales, ponerlos a disposición mediante IMAP, POP3 o JMAP y controlar todo el flujo de mensajes mediante componentes de procesamiento combinables libremente. Por ello, el proyecto se describe no solo como un servidor, sino como una **plataforma de Inversion of Control componible modularmente sobre la JVM** ([Apache James – visión general del proyecto](https://james.apache.org/)).

Esta doble función distingue a James de los Mail Transfer Agents clásicos como Postfix y de las appliances de seguridad terminadas. Un administrador puede operar James como un simple relay SMTP, como un servidor completo de buzones o como un motor de correo integrado en un producto. La comprobación de spam, el cifrado, el archivado o el enrutamiento específico de un dominio no se originan en un bloque de funciones rígido, sino en una canalización de **Matchers** y **Mailets**. Esto hace que James sea extraordinariamente adaptable, pero traslada parte de la responsabilidad sobre el producto del fabricante a la organización operadora.

La explicación sigue un mensaje a través de James: desde los servidores de protocolo, pasando por la cola y la canalización de Mailets, hasta el almacenamiento del buzón. Sobre esta base se presentan las variantes operativas, el diagnóstico y, por último, la evolución técnica del proyecto.

## Clasificación: MTA, MDA y plataforma de aplicaciones

En el sistema de correo electrónico, no todos los componentes desempeñan la misma función. Un **Mail User Agent** (MUA) es el cliente del usuario, por ejemplo Thunderbird. Un **Mail Transfer Agent** (MTA) transporta mensajes entre sistemas. Un **Mail Delivery Agent** (MDA) deposita un mensaje en el buzón de destino. James puede ser MTA y MDA al mismo tiempo; mediante sus módulos de protocolo y buzón también proporciona servicios del lado del servidor para los MUA. La visión general oficial de componentes enumera proyectos separados para servidor, protocolos, Mailets, buzones y pruebas ([Apache James – Software Components](https://james.apache.org/documentation.html)).

| Función | Implementación en James | Punto de transferencia |
|---|---|---|
| Transporte de mensajes | Servidor SMTP y LMTP, cola, Mailet de entrega remota | otros MTA, relays y gateways |
| Entrega local | Canalización de Mailets y Mailbox API | usuarios, dominios y cuotas |
| Acceso al buzón | IMAP, POP3 y JMAP | clientes de correo y aplicaciones web |
| Lógica de filtrado | Matchers, Mailets, Processors y Sieve | reglas internas y servicios externos de comprobación |
| Administración | WebAdmin REST API, CLI, Health Checks y métricas | automatización y monitorización |

Por tanto, James **no es un cliente de correo** ni un Secure Mail Gateway preconfigurado. Proporciona componentes para transporte, entrega, almacenamiento y procesamiento. Que de ello resulte un relay simple, un servicio de correo multiinquilino o un gateway específico de un producto depende de la distribución y configuración elegidas.

## Protocolos, TLS y puertos

James proporciona SMTP, LMTP, IMAP, POP3 y ManageSieve como servicios basados en TCP; JMAP y WebAdmin utilizan HTTP ([Apache James – Protocol Servers](https://james.apache.org/server/feature-protocols.html)). Dependiendo del listener, TLS protege una conexión cifrada desde el inicio o se incorpora a una sesión existente mediante StartTLS. DNS no forma parte del proceso James, pero es indispensable para un MTA público: los registros MX determinan el destino, los registros A y AAAA sus direcciones y los registros PTR influyen en la reputación de las conexiones salientes.

El número de puerto por sí solo no describe la semántica de seguridad. El puerto 25 está destinado al transporte de servidor a servidor; la entrega autenticada por parte de clientes corresponde, según [RFC 6409](https://datatracker.ietf.org/doc/html/rfc6409), al puerto 587. Desde [RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314), el puerto 465 vuelve a estar registrado para Message Submission con cifrado implícito. Para IMAP y POP3 se aplican los mismos dos patrones: conexión en texto claro con posible StartTLS o establecimiento inmediato de TLS.

| Servicio | Puertos típicos | Estándar | Significado en James |
|---|---:|---|---|
| SMTP | 25, 587, 465 | [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321) | aceptación, relay y submission |
| LMTP | configurable, registrado 24 | [RFC 2033](https://datatracker.ietf.org/doc/html/rfc2033) | entrega local con estado por destinatario |
| IMAP4rev2 | 143, 993 | [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051) | acceso síncrono al buzón |
| POP3 | 110, 995 | [RFC 1939](https://datatracker.ietf.org/doc/html/rfc1939) | recuperación sencilla de mensajes |
| ManageSieve | 4190 | [RFC 5804](https://datatracker.ietf.org/doc/html/rfc5804) | gestión de reglas Sieve específicas del usuario |
| JMAP Mail | normalmente 443 | [RFC 8621](https://datatracker.ietf.org/doc/html/rfc8621) | acceso a buzones basado en HTTP para clientes modernos |

Los puertos son configurables; lo vinculante es la combinación de listener, protocolo, modo TLS y autenticación. El [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) sigue siendo la referencia para las asignaciones registradas.

## Enfoque arquitectónico

James sigue una **arquitectura basada en componentes**. Los servidores de protocolo, la cola, la lógica de procesamiento, el buzón, la gestión de usuarios, el índice de búsqueda y la administración están separados entre sí mediante API y se componen mediante inyección de dependencias. Las distribuciones documentadas para James 3.9 se basan para ello en Google Guice; la configuración con Spring pertenece a una generación anterior. El desacoplamiento no es solo una organización del código: permite utilizar la misma [Mailbox API](https://james.apache.org/mailbox/mailbox-api.html) con distintas capas de persistencia y la misma lógica de Mailets en perfiles de servidor muy diferentes.

La ruta central de datos es asíncrona. Un listener SMTP no tiene que entregar completamente un mensaje aceptado antes de responder a la conexión. Coloca un objeto de correo en una cola; un **Spooler** lo extrae más tarde y lo procesa a través del contenedor de Mailets. De este modo, la cola separa la carga de recepción, el tiempo de procesamiento y la disponibilidad de los sistemas posteriores. La documentación de operación distribuida la describe acertadamente como componente obligatorio de un servidor SMTP ([Apache James – Distributed Server Operations](https://james.apache.org/server/manage-guice-distributed-james.html)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 976" src="/images/apache-james-architektur.svg?v=20260813" title="Interaktive Infografik: technische Architektur und Nachrichtenfluss von Apache James" loading="lazy">
  <a href="/images/apache-james-architektur.svg?v=20260813">Abrir la infografía sobre la arquitectura técnica</a>
</iframe>

### La ruta de procesamiento de un mensaje

1. **Aceptación por protocolo:** SMTP o LMTP comprueba la sesión, la autenticación, el remitente del sobre y los destinatarios. Tras el final de `DATA` se crea un objeto interno `Mail` con sobre, contenido MIME y atributos.
2. **Cola:** El objeto se encola de forma persistente o volátil. Solo a partir de este punto la aceptación queda desacoplada del procesamiento.
3. **Spooler:** Los workers extraen entradas de la cola y las transfieren al contenedor de Mailets.
4. **Processor:** Un Processor con nombre contiene una lista ordenada de pares Matcher/Mailet. El Processor obligatorio `root` constituye el punto de entrada.
5. **Matcher:** Un Matcher no modifica el mensaje, sino que devuelve el subconjunto de destinatarios para el que se cumple una condición.
6. **Mailet:** El Mailet correspondiente modifica el mensaje o el sobre, desencadena un efecto secundario, entrega local o remotamente, o se ramifica hacia otro Processor.
7. **Resultado:** El mensaje termina en un buzón de usuario, en la entrega saliente, en un Mail Repository para tratamiento posterior o queda finalizado tras una acción correcta.

Un detalle importante es la **división por destinatario**. Si un Matcher coincide solo con una parte de los destinatarios, el contenedor divide el procesamiento en conjuntos de destinatarios coincidentes y no coincidentes. Por tanto, las reglas no se aplican necesariamente a un mensaje MIME completo. Además, un Mailet puede saltar directamente a otro Processor mediante `ToProcessor`; por ello, la canalización se parece más a un grafo de procesamiento dirigido que a una única lista lineal. La [documentación oficial del contenedor de Mailets](https://james.apache.org/server/feature-mailetcontainer.html) describe exactamente este modelo.

Un patrón mínimo y simplificado tiene este aspecto:

```xml
<processor state="root" enableJmx="true">
  <mailet match="RelayLimit=30" class="ToRepository">
    <repositoryPath>cassandra://var/mail/relay-denied/</repositoryPath>
  </mailet>
  <mailet match="RecipientIsLocal" class="LocalDelivery" />
  <mailet match="All" class="RemoteDelivery" />
</processor>
```

El orden forma parte de la semántica. Una regla de coincidencia amplia al principio puede hacer inalcanzables las reglas posteriores; un bucle infinito entre Processors puede bloquear el Spooler. Por ello, James ofrece un comportamiento de error configurable por Matcher y Mailet, así como Processors de error propios ([Mailet Container Configuration](https://james.apache.org/server/config-mailetcontainer.html)).

La arquitectura de componentes se concreta en cuanto un mensaje alcanza la cola. Entonces, Processor, Matcher y Mailet determinan qué pasos de procesamiento siguen y a dónde llega el resultado.

## Estructura técnica

La arquitectura describe la ruta del mensaje; para la instalación y la operación debe convertirse ahora en una imagen concreta de componentes. Lo decisivo es qué tiempo de ejecución, almacenes y servicios adicionales requiere realmente el perfil de James elegido.

### Stack tecnológico y visión general para administradores

Para una primera clasificación del producto, los límites operativos importan más que los nombres de las clases. La siguiente visión general condensa el stack en las cuestiones que deben aclararse antes de instalar, integrar o asumir un entorno existente:

| Área | Tecnología o artefacto | Lo que debe saber el administrador |
|---|---|---|
| Tiempo de ejecución | Java 21, JVM; código fuente principalmente Java, módulos individuales en Scala | Heap, recolección de basura, threading y parches de JVM forman parte de la operación del servidor |
| Build y paquete | Proyecto Maven multimódulo; ZIP e imágenes de Docker | Los Mailets propios deben ser compatibles con la generación de James, Java y Jakarta |
| Wiring | Guice en la generación 3.9; Spring en instalaciones antiguas | La distribución elegida determina los módulos y archivos de configuración disponibles |
| Configuración | `conf/*.xml`, `conf/*.properties`, variables de entorno | Especialmente importantes: `smtpserver.xml`, `mailetcontainer.xml`, `webadmin.properties`, archivos de JMAP y del backend |
| Procesamiento | MailQueue, Spooler, Processor, Matcher, Mailet | Aceptación, procesamiento y entrega final son estados separados |
| Datos | PostgreSQL/JPA o Cassandra; opcionalmente S3, OpenSearch, RabbitMQ | Fuente, proyección, cola y contenido Blob requieren planes de recuperación separados |
| Administración | WebAdmin REST API y `james-cli` | REST es más potente; CLI está incluido en todas las variantes de wiring |
| Observabilidad | Health Checks, Dropwizard Metrics, Prometheus, JMX, registros, Grafana | Cola, Mailets, Matchers, protocolos y backends poseen métricas propias |
| Seguridad | Keystores TLS, SMTP AUTH, JWT para WebAdmin, segmentación de red | WebAdmin sin JWT activado no está protegido por defecto |

Según el proyecto, todos los archivos de configuración se encuentran en `conf` o `conf/META-INF`; cuáles se aplican realmente depende del wiring y del backend. Los valores pueden obtenerse del entorno mediante `${env:VARIABLE}` ([Apache James – Configuration](https://james.apache.org/server/config.html)). Esto resulta práctico para contenedores, pero no sustituye a la gestión de secretos: los certificados, las claves privadas, las claves JWT y las contraseñas de bases de datos deben proporcionarse como secretos montados o mediante la plataforma de orquestación.

### Capa de protocolo

El proyecto Protocols proporciona implementaciones de servidor extensibles para SMTP, LMTP, IMAP, POP3, ManageSieve y JMAP ([James Protocols](https://james.apache.org/server/feature-protocols.html)). Los listeners no están cableados de forma fija a un almacenamiento determinado. IMAP y JMAP acceden mediante la Mailbox API; SMTP transfiere los mensajes aceptados a la cola y al contenedor de Mailets. Esto permite escalar o desactivar protocolos independientemente de la topología del backend.

### Buzón, Mail Repository y almacenamiento Blob

James distingue tres conceptos de almacenamiento que no deben confundirse durante la operación:

| Almacenamiento | Contenido | Visibilidad | Restauración típica |
|---|---|---|---|
| **Mailbox** | Carpetas, mensajes, flags, UID, ACL y cuotas de un usuario | IMAP/JMAP/POP3 | Restauración o replicación del backend de buzones |
| **Mail Repository** | Mensajes de rutas de procesamiento como `error`, `relay-denied` o cuarentena | solo administración | Corregir la causa y volver a procesar el mensaje |
| **Blob Store** | Contenido MIME binario u objetos grandes | referenciado indirectamente mediante metadatos | Copia de seguridad coherente con metadatos y referencias |

La [documentación de persistencia](https://james.apache.org/server/feature-persistence.html) recalca que un Mail Repository **no** es el buzón de usuario. Esta separación es valiosa para la respuesta a incidentes: un mensaje defectuoso puede aislarse, investigarse y volver a introducirse en la canalización tras una corrección, sin sortear el modelo de buzón.

### Bus de eventos, búsqueda y proyecciones

Las operaciones de buzón generan eventos, como `MailboxAdded`, `MessageMoveEvent`, `FlagsUpdated` o cambios de cuota. Los listeners actualizan a partir de ellos cuotas, índices de búsqueda y otras proyecciones. En el perfil distribuido, RabbitMQ asume la comunicación, OpenSearch la búsqueda y Cassandra los metadatos; el contenido binario se almacena en un Object Store compatible con S3. Esta descomposición permite el escalado horizontal, pero genera **consistencia eventual** entre la fuente y las proyecciones. Los eventos de listener fallidos llegan a un Event Dead Letter y deben supervisarse y, si procede, volver a entregarse ([Distributed James – Mailbox Event Bus](https://james.apache.org/server/manage-guice-distributed-james.html#Mailbox_Event_Bus)).

### MIME, Sieve y autenticación de remitentes

El proyecto James abarca más que el servidor. **Apache Mime4J** analiza estructuras MIME en streaming o como modelo de objetos; **jSieve** implementa el lenguaje de filtrado Sieve; **jSPF** y **jDKIM** proporcionan bibliotecas Java para la comprobación de remitentes y la firma y verificación DKIM, respectivamente. Estos módulos son proyectos independientes y también pueden utilizarse fuera de un servidor James completo ([Apache James – Componentes](https://james.apache.org/documentation.html)).

Que estos componentes se ejecuten en un nodo o distribuidos no es una mera cuestión de rendimiento. La elección también determina la consistencia, el reinicio y el número de backends que deben supervisarse.

## Variantes operativas y escalado

Para James 3.9.0, Apache documenta varios perfiles. No se trata simplemente de instaladores diferentes, sino de distintos modelos de consistencia, escalado y operación. En este estado, la variante JPA se califica explícitamente como **legacy**; junto a ella existen una distribución PostgreSQL y otra distribuida ([Apache James – Downloads](https://james.apache.org/download.cgi)). Los puntos identificados en el gráfico como **derivación operativa** son recomendaciones deducidas de ello y no afirmaciones literales del fabricante.

<iframe class="kb-infographic" style="aspect-ratio: 1280 / 956" src="/images/apache-james-betriebsmodelle.svg?v=20260813" title="Interaktive Infografik: Apache-James-Betriebsmodelle und Technologiestacks" loading="lazy">
  <a href="/images/apache-james-betriebsmodelle.svg?v=20260813">Abrir la infografía comparativa de los modelos operativos</a>
</iframe>

| Perfil | Persistencia y servicios | Adecuado para | Consecuencia operativa |
|---|---|---|---|
| JPA/Guice (legacy) | Base de datos H2 integrada o base de datos SQL externa; modelo clásico de servidor único | laboratorio, migración de instalaciones antiguas, pequeñas soluciones especiales | pocos componentes, pero recorrido estratégico limitado y escalado vertical |
| PostgreSQL | PostgreSQL como núcleo; opcionalmente OpenSearch, RabbitMQ y almacenamiento compatible con S3 | instalaciones nuevas de uno o varios nodos con base relacional | Backup y HA son bien conocidos; introducir servicios adicionales solo cuando se necesite escalado |
| Distributed/Guice | Cassandra, RabbitMQ, OpenSearch y Object Store compatible con S3 | servicios grandes y escalables horizontalmente | varias áreas de fallo, proyecciones, Dead Letters y controles de consistencia más complejos |
| Memory | Componentes In-Memory volátiles | pruebas y desarrollo | sin conservación productiva de datos |

La versión 3.9 destaca la implementación PostgreSQL de alto rendimiento como novedad esencial y la describe como apta tanto para standalone como para escalado mediante RabbitMQ, OpenSearch y S3 ([Apache James 3.9.0](https://james.apache.org/james/update/2025/09/25/james-3.9.0.html)). Para instalaciones nuevas, este suele ser el punto de partida más comprensible: primero consistencia relacional y procedimientos de copia de seguridad conocidos; después, servicios adicionales solo para requisitos medidos concretamente.

## Modelo de seguridad

James proporciona TLS, autenticación SMTP, controles de protocolo y Mailets criptográficos. Sin embargo, esto no implica automáticamente una operación de producción segura. El cifrado de transporte protege un salto; no sustituye ni el cifrado de extremo a extremo ni una comprobación vinculante del destinatario. La [configuración de TLS](https://james.apache.org/server/config-ssl-tls.html) separa el keystore, las cipher suites activadas, StartTLS y TLS implícito por listener. Por ello, un cambio de certificado debe verificarse por separado para SMTP, IMAP, POP3 y HTTP.

**WebAdmin** merece atención especial. La REST API puede modificar dominios, usuarios, buzones, colas, repositorios, cuotas y tareas de mantenimiento. Según la [documentación de WebAdmin](https://james.apache.org/server/manage-webadmin.html), la autenticación JWT está desactivada por defecto; por tanto, sin protección adicional la API nunca debe ser accesible desde una red no controlada. Los endpoints de salud y la documentación de la API también pueden estar deliberadamente fuera de la autenticación.

Un endurecimiento mínimo para producción incluye:

- vincular WebAdmin a una red de gestión, activar JWT y limitar adicionalmente el acceso mediante firewall o reverse proxy;
- impedir relays abiertos mediante reglas explícitas de relay, autenticación y destinatarios;
- operar Submission y SMTP de servidor a servidor en listeners separados con políticas diferentes;
- eliminar dominios de demostración, usuarios de ejemplo y contraseñas predeterminadas de las imágenes de contenedor antes del primer inicio externo;
- gestionar las claves privadas fuera de la capa del contenedor y supervisar las fechas de caducidad;
- tratar los Mailets personalizados como código de aplicación: comprobar dependencias, ejecutar pruebas y limitar permisos de ejecución;
- diseñar conscientemente la comprobación de spam y malware. James es una plataforma; los escáneres externos y servicios de reputación se integran mediante Mailets o transferencias de protocolo.

Para la resolución de problemas, la ruta del mensaje vuelve a comprobarse en el mismo orden: listener, cola, canalización de Mailets, repositorio, buzón y entrega saliente.

## Operación y resolución de problemas

En un servidor de correo modular, «el servicio está en ejecución» no es una afirmación suficiente sobre el estado. Los Health Checks de WebAdmin distinguen `healthy`, `degraded` y `unhealthy`; en modo estricto, incluso un componente degradado da lugar a HTTP 503. Según el perfil, se comprueban, entre otros, JPA o Cassandra, OpenSearch, RabbitMQ, el ciclo de vida de Guice, Event Dead Letters y una entrega de prueba completa ([WebAdmin Health Checks](https://james.apache.org/server/manage-webadmin.html#HealthCheck)).

Para el diagnóstico, un enfoque por capas es más eficiente que una búsqueda global en registros:

1. **Conexión:** ¿El cliente alcanza el listener correcto y TLS funciona con el certificado y nombre de host esperados?
2. **Transacción SMTP:** ¿Qué código de respuesta se entregó para `MAIL FROM`, `RCPT TO` y `DATA`? Un `250` tras `DATA` significa aceptación, no necesariamente entrega final.
3. **Cola:** ¿Aumenta el número de entradas pendientes, crece su antigüedad o se repite el mismo error remoto?
4. **Canalización de Mailets:** ¿Qué Processor y qué par Matcher/Mailet procesaron el mensaje? El ID de correo sirve como clave de correlación.
5. **Repositorio:** ¿Está el mensaje en `error`, `address-error`, `relay-denied` o en un repositorio propio? Corregir primero la causa y luego reprocesar.
6. **Buzón y eventos:** ¿El mensaje está presente en el almacén principal de buzones, pero falta en el índice de búsqueda o JMAP? Entonces los listeners, Dead Letters y la reindexación son más relevantes que SMTP.
7. **Remote Delivery:** Para la entrega saliente, comprobar por separado DNS, ruta, TLS, código de la contraparte, plan de reintentos y generación de rebotes.

Una comprobación sintética compacta puede conectar el plano administrativo y el de datos:

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für den Health Check">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$headers = @{ Authorization = "Bearer $env:JAMES_ADMIN_JWT" }
Invoke-RestMethod `
  -Uri "https://james-admin.example.net/healthcheck?strict" `
  -Headers $headers</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --fail --silent \
  -H "Authorization: Bearer $JAMES_ADMIN_JWT" \
  "https://james-admin.example.net/healthcheck?strict"</code></pre>
  </div>
</div>

En Windows, [`Invoke-RestMethod`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod) llama al endpoint REST; en Linux y Unix, [`curl`](https://curl.se/docs/manpage.html) realiza la misma comprobación HTTP. Ambos comandos prueban aquí exclusivamente el Health Check documentado de WebAdmin y no sustituyen una transacción sintética SMTP o de buzón.

Además, se deberían alertar como mínimo la profundidad y antigüedad de la cola, los repositorios de errores, los Event Dead Letters, el retraso de indexación de OpenSearch, las latencias de backend, las clases de respuesta SMTP, la memoria JVM y la vigencia de los certificados. En la variante distribuida, un proceso James en verde con RabbitMQ u OpenSearch perturbado es solo un éxito parcial.

### Herramientas para el puesto de trabajo del administrador

James incluye un cliente de línea de comandos para dominios, usuarios, buzones, mappings, cuotas y reindexación; en contenedores Guice está disponible como `james-cli` ([James CLI](https://james.apache.org/server/manage-cli.html)). Para un diagnóstico fiable, además deben estar disponibles algunas herramientas independientes del protocolo en el puesto de trabajo del administrador:

| Herramienta | Uso con James |
|---|---|
| [`swaks`](https://www.jetmore.org/john/code/swaks/) | transacción SMTP y Submission completa con AUTH, TLS, sobre y cabeceras configuradas libremente |
| [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) | comprobar cadena de certificados, SNI, cipher y StartTLS en SMTP, IMAP o POP3 |
| [`curl`](https://curl.se/docs/manpage.html) y [`jq`](https://jqlang.org/manual/) | consultar automáticamente WebAdmin, Health Checks, tareas y métricas |
| [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) o [`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) | comprobar MX, A/AAAA, PTR, SPF, DKIM y DMARC |
| [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) o [Wireshark](https://www.wireshark.org/docs/wsug_html_chunked/) | diferenciar handshake, retransmisiones, interrupciones de conexión y diálogos de protocolo |
| [Prometheus](https://prometheus.io/docs/introduction/overview/) y [Grafana](https://grafana.com/docs/grafana/latest/) | observar métricas de cola y protocolo, percentiles de latencia, tiempos de ejecución de Mailet/Matcher y estados de backend |
| [JMX](https://docs.oracle.com/en/java/javase/21/management/java-management-extensions-jmx-user-guide.html), [VisualVM](https://visualvm.github.io/documentation.html) y [`jcmd`](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jcmd.html) | investigar heap, threads, recolección de basura y métricas internas de la JVM |

La [documentación nativa de métricas](https://james.apache.org/server/metrics.html) enumera, entre otras, conexiones SMTP, IMAP y LMTP activas, entradas de cola, mensajes enviados y entregados, tiempos de respuesta por protocolo y tiempos de ejecución de Mailets y Matchers individuales. Estas métricas son más significativas que una única disponibilidad del proceso, porque reflejan la ruta de un mensaje a través de la arquitectura.

## Historia técnica

James no surgió como un port de un MTA Unix existente. Las páginas de proyecto conservadas más antiguas, de **1997/1998**, describen inicialmente un servidor Java previsto, todavía no utilizable, basado en paquetes comunes del Java Apache Project. Se preveían una interfaz de protocolo común, almacenamiento JDBC y una interfaz **MailServlet** inspirada en Servlets; la infraestructura del entorno Apache JServ sirvió como trabajo técnico previo ([Archivo de James-1.0](https://svn.apache.org/repos/asf/james/server/tags/james_1_0/docs/index.html?p=1400000)). La posterior Mailet API conservó la idea fundamental de pequeños componentes de procesamiento desplegables, sin convertirse en parte de la especificación Java Servlet.

| Período | Hito de desarrollo técnico |
|---|---|
| 1997–1998 | Diseño en el Java Apache Project: servidor Java puro, interfaces comunes de protocolo y recursos, idea de MailServlet |
| Febrero de 2001 | Migración del Java Apache Project al proyecto Jakarta ([Jakarta News 2001](https://jakarta.apache.org/site/news/news-2001.html#20010311.1)) |
| James 1.x/2.x | servidor SMTP/POP3 estable, NNTP temporalmente; motor de Mailets, almacenamiento de archivos y RDBMS; contenedor de componentes Avalon/Phoenix ([Archivo de documentación](https://james.apache.org/server/archive/document_archive.html)) |
| primeros años 2000 | ascenso de subproyecto Jakarta a proyecto Top-Level independiente de la Apache Software Foundation ([James 2.1.3 – página de proyecto archivada](https://svn.apache.org/repos/asf/james/server/tags/deprecated/build_2_2_0_RC1/www/index.html?p=1400000)) |
| 2010 | James 3.0 M1 con compatibilidad IMAP completa, SMTP/LMTP, Mailet API revisada y almacenamiento Maildir, JPA y JCR ([Anuncio de lanzamiento](https://james.apache.org/james/update/2010/11/05/james-3.0-M1.html)) |
| James 3.x | sustitución de Avalon/Phoenix por Spring y posterior orientación estratégica hacia Guice; ampliación de IMAP, JMAP, administración REST y backends distribuidos |
| Septiembre de 2025 | James 3.9.0: cambio de `javax` a `jakarta`, Java 21 y nueva implementación PostgreSQL ([Anuncio de lanzamiento](https://james.apache.org/james/update/2025/09/25/james-3.9.0.html)) |

El código fuente se encuentra en el repositorio oficial [apache/james-project](https://github.com/apache/james-project). La generación 3.9 examinada aquí está compuesta principalmente por Java; algunos módulos utilizan Scala. Se compila como un gran proyecto Maven multimódulo. La larga historia de desarrollo explica por qué varias generaciones son visibles simultáneamente en la documentación y las instalaciones: términos de Phoenix y Spring en textos antiguos, Guice en la documentación 3.x, JPA como ruta legacy y perfiles PostgreSQL o Cassandra para despliegues distribuidos.

## Idoneidad y límites

James resulta especialmente adecuado cuando el correo electrónico es **parte de una aplicación** y no solo infraestructura: procesamiento basado en reglas, Mailets propios, protocolos abiertos, JMAP, almacenamiento de datos controlable o escalado horizontal sin un núcleo de servidor propietario. Las API públicas permiten seguir desarrollando por separado el transporte, el buzón y la lógica de negocio.

James es menos adecuado para organizaciones que esperan una appliance llave en mano con GUI completa, protección antispam y antimalware preconfigurada, SLA del fabricante y un único objeto de copia de seguridad. La libertad modular genera trabajo de integración. En especial, el perfil distribuido exige experiencia operativa con varios sistemas de datos y una definición clara de fuente, proyección, reconstrucción y punto de recuperación.

Por ello, la cuestión arquitectónica decisiva es: **¿debe operarse el correo electrónico como un sistema de protocolos configurable o como un producto terminado?** Para el primer caso, James proporciona un conjunto de herramientas abierto e inusualmente profundo. Para el segundo, un producto más preconfigurado suele ser más económico.

## Fuentes

- [Apache Projects – James Committee](https://projects.apache.org/committee.html?james)
- [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)
- [Microsoft Learn – Invoke-RestMethod](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/invoke-restmethod)
- [curl – Manpage](https://curl.se/docs/manpage.html)
- [SWAKS – Swiss Army Knife for SMTP](https://www.jetmore.org/john/code/swaks/)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [jq – Manual](https://jqlang.org/manual/)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [tcpdump – Manpage](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [Wireshark – User’s Guide](https://www.wireshark.org/docs/wsug_html_chunked/)
- [Prometheus – Overview](https://prometheus.io/docs/introduction/overview/)
- [Grafana – Documentation](https://grafana.com/docs/grafana/latest/)
- [Oracle – JMX User Guide](https://docs.oracle.com/en/java/javase/21/management/java-management-extensions-jmx-user-guide.html)
- [VisualVM – Documentation](https://visualvm.github.io/documentation.html)
- [Oracle – jcmd](https://docs.oracle.com/en/java/javase/21/docs/specs/man/jcmd.html)
- [Apache James – visión general del proyecto](https://james.apache.org/) – autodescripción, JVM, protocolos, módulos y objetivos arquitectónicos.
- [Apache James – Software Components](https://james.apache.org/documentation.html) – subproyectos de servidor, Mailet, Mailbox, Protocols y otros.
- [Apache James – Protocol Servers](https://james.apache.org/server/feature-protocols.html) – servicios de protocolo compatibles.
- [RFC 6409](https://datatracker.ietf.org/doc/html/rfc6409)
- [RFC 8314](https://datatracker.ietf.org/doc/html/rfc8314)
- [IETF: SMTP](https://datatracker.ietf.org/doc/html/rfc5321), [LMTP](https://datatracker.ietf.org/doc/html/rfc2033), [Message Submission](https://datatracker.ietf.org/doc/html/rfc6409), [IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051), [POP3](https://datatracker.ietf.org/doc/html/rfc1939), [ManageSieve](https://datatracker.ietf.org/doc/html/rfc5804) y [JMAP Mail](https://datatracker.ietf.org/doc/html/rfc8621) – estándares normativos de protocolo.
- [RFC 2033](https://datatracker.ietf.org/doc/html/rfc2033)
- [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051)
- [RFC 1939](https://datatracker.ietf.org/doc/html/rfc1939)
- [RFC 5804](https://datatracker.ietf.org/doc/html/rfc5804)
- [RFC 8621](https://datatracker.ietf.org/doc/html/rfc8621)
- [IANA Service Name and Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – puertos registrados.
- [Mailbox API](https://james.apache.org/mailbox/mailbox-api.html)
- [Apache James – Managing Distributed James](https://james.apache.org/server/manage-guice-distributed-james.html) – Cassandra, S3, OpenSearch, RabbitMQ, bus de eventos y operación.
- [Apache James – Mailet Container](https://james.apache.org/server/feature-mailetcontainer.html) – Matchers, Mailets, Processors, Spooler y división de destinatarios.
- [Apache James – Mailet Container Configuration](https://james.apache.org/server/config-mailetcontainer.html) – configuración y tratamiento de errores de la canalización.
- [Apache James – Configuration](https://james.apache.org/server/config.html) – directorio de configuración, archivos y variables de entorno.
- [Apache James – Persistence](https://james.apache.org/server/feature-persistence.html) – delimitación entre Mailbox y Mail Repository.
- [Apache James – Downloads](https://james.apache.org/download.cgi) – perfiles de servidor y descargas oficiales.
- [Apache James Server 3.9.0](https://james.apache.org/james/update/2025/09/25/james-3.9.0.html) – Java 21, migración a Jakarta e implementación PostgreSQL.
- [Apache James – SSL/TLS Configuration](https://james.apache.org/server/config-ssl-tls.html) – modos TLS y configuración de listeners.
- [Apache James – WebAdmin](https://james.apache.org/server/manage-webadmin.html) – administración REST, nota sobre JWT y Health Checks.
- [Apache James – Command Line](https://james.apache.org/server/manage-cli.html) – CLI para dominios, usuarios, buzones, mappings, cuotas y reindexación.
- [Apache James – Metrics](https://james.apache.org/server/metrics.html) – Prometheus, JMX y métricas operativas disponibles.
- [Archivo James-1.0 del Java Apache Project](https://svn.apache.org/repos/asf/james/server/tags/james_1_0/docs/index.html?p=1400000) – planificación temprana de arquitectura y MailServlet.
- [Jakarta Project News 2001](https://jakarta.apache.org/site/news/news-2001.html) – migración del proyecto James a Jakarta.
- [Apache James Document Archive](https://james.apache.org/server/archive/document_archive.html) – documentación de las versiones 1.x y 2.x.
- [James 2.1.3 – página de proyecto archivada](https://svn.apache.org/repos/asf/james/server/tags/deprecated/build_2_2_0_RC1/www/index.html?p=1400000)
- [Apache James 3.0 M1](https://james.apache.org/james/update/2010/11/05/james-3.0-M1.html) – IMAP, perfiles de almacenamiento y Mailet API de la generación 3.x.
- [Apache James – repositorio GitHub](https://github.com/apache/james-project) – código fuente, build y estructura de módulos.
