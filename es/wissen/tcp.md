---
title: "TCP: estados, flujo de datos y diagnóstico"
blatt: "tcp"
description: "TCP como base de transporte para servicios de mensajería y directorio: modelo de sockets, estados de conexión, números de secuencia, confirmaciones, control de flujo y congestión, temporizadores, patrones de error típicos y herramientas de diagnóstico para administradores."
fakten:
  - label: Nombre completo
    wert: Transmission Control Protocol
    href: https://datatracker.ietf.org/doc/html/rfc9293
  - label: Estándar
    wert: RFC 9293 · STD 7
    href: https://www.rfc-editor.org/info/rfc9293
  - label: Modelo de servicio
    wert: flujo de bytes fiable, ordenado y bidireccional
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-2
  - label: Punto final
    wert: dirección IP y número de puerto de 16 bits
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.1
  - label: Conexión
    wert: par único de socket local y remoto
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.1
  - label: Cabecera
    wert: al menos 20 bytes más opciones
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.1
  - label: Fiabilidad
    wert: números de secuencia, ACK acumulativos y retransmisión
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.3
  - label: Control de flujo
    wert: ventana de recepción rwnd
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-3.8.4
  - label: Control de congestión
    wert: cwnd · Slow Start · Congestion Avoidance
    href: https://datatracker.ietf.org/doc/html/rfc5681
  - label: Rangos de puertos
    wert: 0–1023 · 1024–49151 · 49152–65535
    href: https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml
  - label: Cifrado
    wert: ninguno; protección mediante TLS, IPsec o TCP-AO
    href: https://datatracker.ietf.org/doc/html/rfc9293#section-7
  - label: Diagnóstico
    wert: prueba de puertos · estado de socket · captura de paquetes
    href: https://www.tcpdump.org/manpages/tcpdump.1.html
werbung:
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: 6c749be6d80debccf17ceb4386ba0e298c11deedf855db326e7d20da9593fc05
translationModel: gpt-5.6-terra
translatedAt: 2026-10-14T09:36:02.940Z
translationReview: automatic
---

# TCP: estados, flujo de datos y diagnóstico

TCP transforma el transporte de paquetes no fiable de IP en un flujo de datos bidireccional y ordenado para las aplicaciones. La aplicación escribe bytes en un socket; TCP los segmenta, numera su posición, confirma los rangos recibidos y retransmite las lagunas. SMTP, IMAP, POP3, LDAP y HTTPS se basan en esta garantía. Por ello, cuando una aplicación de este tipo «se bloquea», el administrador debe distinguir primero si TCP sigue esperando una confirmación o si ya está bloqueado el protocolo de aplicación.

El artículo sigue el ciclo de vida de una conexión: socket y listener, handshake de tres vías, transferencia de datos, control de flujo y congestión, cierre y TIME-WAIT. A partir de este proceso normal se derivan después el diagnóstico, la planificación de capacidad, la monitorización y el endurecimiento.

Sin embargo, TCP no conoce mensajes, usuarios, certificados ni direcciones de correo. Tampoco conserva los límites de las aplicaciones: dos operaciones de escritura consecutivas pueden agruparse en un segmento o distribuirse entre varios segmentos. Por tanto, un handshake TCP correcto solo demuestra que dos extremos TCP pudieron establecer una conexión. Que después aparezca un banner SMTP, se negocie TLS o se complete un enlace LDAP depende de la aplicación subyacente.

Para los administradores, TCP es relevante menos como modelo de capas que como **sistema de estados y señales**. SYN, ACK, FIN y RST indican en qué punto de transferencia falla una conexión. Los estados de socket, las retransmisiones, las ventanas de recepción y los temporizadores explican por qué un servicio puede ser accesible, pero lento o aparentemente congelado.

La explicación sigue una conexión TCP desde el primer SYN hasta el flujo de bytes ordenado y su terminación o restablecimiento. Las ventanas, retransmisiones, congestión y diagnóstico se explican en las fases correspondientes.

## Modelo de sockets y conexiones

Antes de que fluyan los paquetes, una aplicación vincula un socket a una dirección y un puerto. Por ejemplo, un servidor escucha en TCP 25; cada conexión aceptada se identifica de forma única mediante dirección de origen, puerto de origen, dirección de destino y puerto de destino. Por eso miles de clientes pueden usar el mismo listener, siempre que difieran al menos en su dirección de origen o puerto efímero. Por consiguiente, para el diagnóstico no basta con «puerto 25»: siempre se necesita la tupla completa de conexión ([RFC 9293, sección 3.1](https://datatracker.ietf.org/doc/html/rfc9293#section-3.1)).

En un servidor, un proceso vincula un socket a una dirección local y lo pone en estado **LISTEN**. La elección de la dirección de enlace es importante operativamente:

| Enlace | Visibilidad | Consecuencia típica |
|---|---|---|
| `127.0.0.1:25` o `::1:25` | solo host local | La monitorización en el servidor no muestra anomalías; los sistemas remotos no obtienen conexión |
| `0.0.0.0:25` | todas las direcciones IPv4 locales | El listener sigue direcciones IPv4 adicionales, si el firewall y el enrutamiento lo permiten |
| `[::]:25` | comodín IPv6 | El comportamiento dual-stack depende del sistema operativo y de la opción `IPV6_V6ONLY` |
| IP de servicio concreta | solo esta dirección | Separación controlada, pero sensible a cambios de IP o de clúster |

El puerto del servidor suele estar definido por un [registro de IANA](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml); el cliente normalmente elige un puerto efímero del rango gestionado por el sistema operativo. Por tanto, los firewalls, gateways NAT y balanceadores de carga deben permitir no solo el puerto de destino conocido, sino también toda la ruta de retorno hacia el puerto de origen dinámico.

### TCP Control Block

Para cada conexión, la pila TCP mantiene un **Transmission Control Block** (TCB). Contiene, entre otros elementos, el estado, los sockets local y remoto, la secuencia de envío y recepción, ventanas, temporizadores y parámetros de seguridad. El TCB es la razón por la que una conexión tiene estado y por la que los balanceadores de carga o firewalls deben operar con ruta de retorno simétrica: si un segmento de respuesta llega a otro nodo sin el estado correspondiente, este puede descartarlo o responder con RST.

## Establecimiento de conexión, transferencia de datos y cierre

La apertura activa inicia el estado visible de la conexión. El cliente envía SYN junto con el número de secuencia inicial, el listener responde con SYN+ACK y su propio número de secuencia, y después el cliente confirma con ACK. Ya estos tres segmentos negocian opciones como Maximum Segment Size, Window Scale, Selective Acknowledgment y Timestamps. Si falta un paquete, la respuesta llega por otra ruta o un producto intermedio modifica las opciones, el handshake ya muestra dónde se separan la accesibilidad y el rendimiento posterior.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 976" src="/images/kb-interaktiv-tcp-verbindung.svg?v=20260813" title="Interaktive Infografik: TCP-Verbindungslebenszyklus und Diagnosepunkte" loading="lazy">
  <a href="/images/kb-interaktiv-tcp-verbindung.svg?v=20260813">Abrir la infografía sobre el ciclo de vida de una conexión TCP</a>
</iframe>

Después del handshake, la conexión se encuentra en **ESTABLISHED**. Ambos lados pueden enviar datos de forma independiente y cerrar su dirección de envío con FIN. Por ello, un cierre ordenado normalmente requiere un FIN y un ACK por dirección. El lado que envía el último ACK permanece en **TIME-WAIT**, para evitar que segmentos retrasados de una conexión antigua lleguen a una nueva conexión con el mismo par de sockets ([RFC 9293, secciones 3.3.2 y 3.6](https://datatracker.ietf.org/doc/html/rfc9293#section-3.3.2)).

Un **RST** no es un cierre ordenado. Interrumpe la conexión y descarta los estados pendientes. Durante el establecimiento de la conexión, un RST válido normalmente significa que el host de destino es accesible, pero el socket no es aceptado. Tras haberse transferido datos, un RST puede proceder de la aplicación, del sistema operativo o de un nodo intermedio con estado. Por ello, la posición del RST en la captura de paquetes es más importante que el mensaje de error genérico «connection reset».

### Los estados como indicadores operativos

| Estado | Significado técnico | Lo que su acumulación sugiere |
|---|---|---|
| `LISTEN` | el socket pasivo espera SYN | estado esperado del servidor; la ausencia de listener apunta a un problema de servicio o enlace |
| `SYN-SENT` | SYN local enviado, falta la respuesta correspondiente | filtro, ruta, dirección incorrecta o contraparte sobrecargada |
| `SYN-RECEIVED` | SYN aceptado, falta el ACK final | SYN flood, ruta de retorno asimétrica, problema de cliente o firewall |
| `ESTABLISHED` | espacios de secuencia sincronizados | normal; muchas conexiones de larga duración pueden ser deseadas, por ejemplo con IMAP o HTTP/2 |
| `FIN-WAIT-1/2` | el lado local ha cerrado | la contraparte o la aplicación termina su dirección con retraso |
| `CLOSE-WAIT` | FIN remoto recibido, la aplicación local aún no ha cerrado | a menudo una fuga de recursos o un hilo de aplicación bloqueado |
| `LAST-ACK` | FIN local enviado tras el cierre remoto | falta el ACK de la contraparte o la ruta de retorno está perturbada |
| `TIME-WAIT` | la antigua tupla de cuatro elementos se protege antes de reutilizarse | normal en el lado que cierra activamente; problemático solo junto con presión sobre puertos o TCB |

Los nombres de estado están definidos en [RFC 9293](https://datatracker.ietf.org/doc/html/rfc9293#section-3.3.2). Un número elevado no es un error en sí mismo. Son decisivos la antigüedad, la tasa de crecimiento, las contrapartes afectadas y la relación con la tasa de conexiones, los descriptores de archivo, el rango de puertos efímeros y los límites de la aplicación.

## Números de secuencia, ACK y retransmisión

Después del handshake, TCP cuenta el flujo de datos byte a byte. El número de secuencia indica el primer byte de un segmento; el ACK indica el siguiente byte esperado y confirma así todo lo anterior. Los paquetes pueden perderse, llegar duplicados o cambiar de orden. El receptor puede almacenarlos en búfer, pero solo entrega a la aplicación un flujo sin lagunas. Precisamente por eso, una única región ausente puede bloquear una aplicación aparentemente inactiva, aunque paquetes posteriores ya hayan llegado al host.

Si se pierde un segmento o su ACK, el emisor puede detectar la región ausente de dos formas:

1. **Retransmission Timeout (RTO):** Se deriva un temporizador del Round-Trip Time suavizado y de su variación. Cuando expira, se retransmite el segmento no confirmado más antiguo. [RFC 6298](https://datatracker.ietf.org/doc/html/rfc6298) define el cálculo, los valores mínimos y el backoff exponencial.
2. **Fast Retransmit:** Varios ACK duplicados indican una laguna en el flujo de datos. El emisor puede retransmitir la región presumiblemente perdida sin esperar al RTO ([RFC 5681](https://datatracker.ietf.org/doc/html/rfc5681#section-3.2)).

Una retransmisión no demuestra automáticamente pérdida de paquetes en la red. Una captura puede haberse realizado en un solo lado, pueden faltar paquetes por pérdidas de captura o puede mostrar una retransmisión espuria. Wireshark identifica expresamente su análisis TCP como una interpretación derivada del flujo de paquetes observado ([Wireshark User’s Guide – TCP Analysis](https://www.wireshark.org/docs/wsug_html_chunked/ChAdvTCPAnalysis.html)). Por tanto, para una afirmación sólida se deben correlacionar, en lo posible, capturas en el cliente, el servidor y un punto intermedio relevante.

## Control de flujo y control de congestión

La fiabilidad por sí sola no evita que un emisor rápido sobrecargue al receptor o a la red. Por ello, TCP limita los bytes no confirmados simultáneamente mediante dos ventanas, de las cuales al final actúa la menor:

- La **ventana de recepción** `rwnd` protege al receptor. Describe el búfer disponible actualmente. Si el receptor anuncia una Zero Window, el emisor debe esperar, salvo por Window Probes periódicos. Una Zero Window persistente no suele ser un problema de red, sino que indica que la aplicación receptora no lee los datos de su socket con suficiente rapidez.
- La **Congestion Window** `cwnd` protege la ruta. El emisor la mantiene localmente y la ajusta basándose en confirmaciones y señales de pérdida. [RFC 5681](https://datatracker.ietf.org/doc/html/rfc5681) describe Slow Start, Congestion Avoidance, Fast Retransmit y Fast Recovery.

De forma aproximada, es efectiva la menor entre `rwnd` y `cwnd`. Por ello, un ancho de banda elevado por sí solo no garantiza un alto rendimiento. En una ruta con un gran producto ancho de banda-latencia debe permitirse suficiente cantidad de datos no confirmados. La opción Window Scale amplía la ventana de 16 bits; Timestamps admite, entre otras funciones, la medición de RTT y la protección contra números de secuencia antiguos ([RFC 7323](https://datatracker.ietf.org/doc/html/rfc7323)). Ambas opciones se negocian durante el handshake y no pueden activarse posteriormente para la misma conexión.

En transacciones SMTP cortas, suele dominar el número de viajes de ida y vuelta: handshake TCP, handshake TLS opcional y varias respuestas de protocolo. Para adjuntos grandes y flujos de replicación también adquieren relevancia RTT, pérdida, ventanas, offloading y el algoritmo de control de congestión. Por tanto, un problema de rendimiento solo es «ancho de banda» cuando se han excluido el tiempo de handshake, las pausas de aplicación, las Zero Windows y las retransmisiones.

## Temporizadores, keepalives y tiempos de espera inactivos

Cuando no llega una respuesta inmediata, los temporizadores determinan el siguiente paso. El Retransmission Timeout activa repeticiones de datos, el Persist Timer mantiene una conexión pese a una Zero Window y TIME-WAIT evita que segmentos antiguos lleguen a una conexión nueva. Los sistemas operativos y las aplicaciones añaden Keepalive y User Timeout. Quien solo registra «timeout» pierde estas diferencias y, con ellas, la causa real del error.

Los TCP keepalives suelen estar desactivados de forma predeterminada o no comienzan hasta tras largos periodos de inactividad. No sustituyen una comprobación de estado a nivel de aplicación. Un keepalive correcto solo indica que el par TCP responde; no demuestra que un worker SMTP procese mensajes o que pueda atenderse una consulta LDAP. A la inversa, los firewalls y balanceadores de carga suelen desconectar conexiones inactivas antes que los sistemas finales. Entonces, en un lado permanece un socket aparentemente válido hasta que el siguiente intento de datos provoca un timeout o RST.

Por ello, en conexiones IMAP, pools LDAP o API de larga duración deben encajar cuatro valores: el timeout de inactividad de la aplicación, el intervalo TCP keepalive, el timeout de inactividad de cada nodo intermedio y la lógica de reconexión del cliente. El menor timeout no anunciado determina el comportamiento observado.

## Diagnóstico desde el socket hasta el paquete

La búsqueda de errores se vuelve sólida cuando describe la misma conexión que la aplicación: host de origen, nombre de destino, dirección resuelta, puerto de destino, momento y protocolo esperado. Un ping no sustituye estos datos. ICMP puede estar bloqueado aunque TCP funcione; a la inversa, una respuesta Echo no demuestra ni un listener ni el estado de un firewall con estado.

### Accesibilidad y listener

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für den TCP-Verbindungstest">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection -ComputerName mail.example.ch -Port 25 -InformationLevel Detailed

Get-NetTCPConnection -State Listen -LocalPort 25 |
  Select-Object LocalAddress, LocalPort, OwningProcess</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 mail.example.ch 25

sudo ss -ltnp 'sport = :25'</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) combina en Windows la resolución de nombres, la dirección de destino, la prueba de puerto y el diagnóstico de ruta opcional. En Unix, [`nc`](https://man.openbsd.org/nc.1) abre de forma específica conexiones TCP o UDP; [`telnet`](https://man.openbsd.org/telnet.1) también puede abrir un puerto TCP, pero procesa el protocolo Telnet y no es un sustituto completamente transparente. [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) lee información de sockets del kernel Linux y puede filtrar estados, temporizadores, colas y procesos. Una prueba de conexión exitosa abre y cierra una conexión TCP real; por ello, en servicios de producción debe emplearse de forma específica y no como un escaneo de puertos de alta frecuencia.

### Captura de paquetes y filtros de estado

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für den TCP-Paketmitschnitt">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">pktmon filter remove
pktmon filter add MailTcp -t TCP -p 25
pktmon start --capture --pkt-size 0 --file-name C:\Temp\mail-tcp.etl
# Problem reproduzieren, danach:
pktmon stop
pktmon etl2pcap C:\Temp\mail-tcp.etl --out C:\Temp\mail-tcp.pcapng</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sudo tcpdump -i any -nn -s 0 -w /tmp/mail-tcp.pcap \
  'host 192.0.2.25 and tcp port 25'

ss -tin state established '( dport = :25 or sport = :25 )'</code></pre>
  </div>
</div>

[`Pktmon`](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) captura paquetes en la pila de red de Windows y puede convertir su formato ETL a PCAPNG. En Unix, [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) utiliza una expresión de captura y aquí escribe un archivo PCAP para su análisis posterior. Una captura completa conserva la carga útil y puede contener credenciales o contenido de mensajes. Debe tratarse como una exportación de correo: filtrar estrictamente, capturar durante poco tiempo, almacenar de forma protegida y eliminar después del análisis. Con TLS siguen siendo visibles direcciones, puertos, tamaños de paquete, temporización, metadatos del handshake y señales TCP, pero no el contenido cifrado de la aplicación.

### Patrón de señales y punto de transferencia más probable

| Observación | Perspectiva TCP | Siguiente comprobación más probable |
|---|---|---|
| SYN se repite, sin respuesta | el establecimiento de conexión no alcanza un par que responda | destino DNS, ruta, Security Group, descarte de firewall, ruta de retorno asimétrica |
| SYN recibe RST inmediato | la ruta de destino funciona, el socket se rechaza | listener, dirección de enlace, destino NAT, configuración de puerto |
| el handshake se completa, no hay banner | TCP está establecido, la aplicación no entrega datos | worker, thread pool, protocolo proxy, registro de aplicación, dependencia de backend |
| RST inmediatamente después de datos del cliente | el par o una middlebox descarta la sesión | protocolo incorrecto, ACL, TLS en puerto de texto plano, dispositivo de inspección |
| aumentan las retransmisiones y Dup ACK | pérdida, reordenamiento o paquetes de captura ausentes | errores de interfaz, congestión, MTU/PMTUD, cambio de ruta, captura en la contraparte |
| Zero Window | el receptor informa de un búfer lleno | CPU, hilo de aplicación, almacenamiento, Garbage Collection, búfer de socket |
| muchos `CLOSE-WAIT` antiguos | el par ha terminado, la aplicación local no cierra | fuga de descriptores de archivo, ruta de código bloqueada, pool de conexiones |
| los connects fallan solo bajo carga | límite de estado o capacidad | listen backlog, SYN cookies, NAT/Conntrack, puertos efímeros, límite de FD |
| los datos pequeños funcionan, los grandes se bloquean | error de ruta dependiente del tamaño | MTU, filtro PMTUD/ICMP, MSS clamping, límites de registros TLS/proxy |

## Capacidad y dominios de fallo

Tras la conexión individual llega la cuestión de la capacidad. Cada socket ocupa estado en el cliente y el servidor, y con frecuencia también en el firewall, NAT, balanceador de carga o proxy. Por tanto, el número sostenible de conexiones no viene determinado solo por CPU o ancho de banda, sino por la tabla, cola o conjunto de puertos más pequeño a lo largo de la ruta real.

Los administradores deberían distinguir al menos los siguientes recursos:

- **Listen backlog y SYN backlog:** limitan conexiones completamente abiertas y parcialmente abiertas, respectivamente;
- **descriptores de archivo y límites de proceso:** un puerto libre no basta si la aplicación ya no puede aceptar sockets;
- **puertos efímeros:** los relays y proxies salientes solo pueden alcanzar el mismo socket de destino en paralelo con distintos sockets locales;
- **tablas NAT y Conntrack:** mantienen sus propios temporizadores y pueden agotarse pese a que los recursos de los sistemas finales estén libres;
- **memoria por socket:** los búferes de envío/recepción, el estado TLS y el contexto de aplicación se multiplican con el número de conexiones;
- **distribución de TIME-WAIT:** el lado que cierra activamente soporta este estado; por tanto, las decisiones de arquitectura desplazan la carga;
- **afinidad del balanceador de carga y ruta de retorno:** los dispositivos con estado deben ver ambas direcciones de la misma conexión.

Reducir a ciegas los temporizadores de TIME-WAIT, keepalive o retransmisión rara vez resuelve la causa. Modifica la protección del protocolo y la semántica de los errores. Primero deben medirse la tasa de conexiones, el tiempo de mantenimiento, la distribución de tuplas de cuatro elementos y la tabla realmente agotada.

## Monitorización para entornos de mensajería

De estos estados se deriva la monitorización. La utilización de la interfaz por sí sola no detecta handshakes lentos, retransmisiones, tormentas de resets ni un receptor con ventana cerrada. Por ello, un dashboard útil combina la perspectiva del host, la red y la aplicación, y coloca al menos las siguientes series temporales una junto a otra:

| Métrica | Indicación | Alarma solo en relación con |
|---|---|---|
| tasa de éxito de conexión y p95/p99 | accesibilidad y tiempo de establecimiento | destino, familia IP, puerto y ubicación |
| nuevas conexiones por segundo | carga y churn | conexiones establecidas y tiempo medio de mantenimiento |
| `SYN-SENT` / `SYN-RECEIVED` | conexiones parcialmente abiertas | antigüedad, descartes de backlog, SYN cookies |
| RST por dirección y fase de vida | interrupciones activas | antes/después del handshake, nodo emisor, código de aplicación |
| tasa de retransmisiones | señal de pérdida o reordenamiento | RTT, descartes de interfaz, posición de captura |
| RTT y RTO | retraso de ruta y comportamiento del temporizador | región, contraparte, hora del día |
| duración de Zero Window | el receptor no lee | CPU, almacenamiento, JVM/GC, thread pools |
| antigüedad de `CLOSE-WAIT` | la aplicación local no cierra | proceso, versión y pool de conexiones |
| utilización de NAT/Conntrack | estado en el nodo intermedio | timeouts, uso de puertos, ruta de retorno |

Las pruebas sintéticas de puertos deberían acoplarse a una prueba de protocolo. Para SMTP, esto significa: TCP connect, banner, EHLO, StartTLS opcional y una transacción controlada son puntos de medición separados. Solo así se puede reconocer si la ruta de red, TLS o la aplicación de correo causa la latencia.

## Seguridad y endurecimiento

TCP proporciona orden y corrección de errores, pero ni confidencialidad ni identidad criptográfica. Para ello, las aplicaciones suelen añadir [TLS](/kb/tls), las redes IPsec cuando procede y las conexiones TCP especiales TCP-AO. Aunque los números de secuencia dificultan la inyección a ciegas, no sustituyen la autenticación. El endurecimiento también debe considerar SYN floods, resets falsificados, colas demasiado grandes y listeners susceptibles de abuso ([RFC 9293, sección 7](https://datatracker.ietf.org/doc/html/rfc9293#section-7)).

Para servicios de mensajería expuestos son relevantes los siguientes controles:

- publicar únicamente los listeners y familias de direcciones realmente necesarios;
- modelar por separado los roles de servidor y cliente en los firewalls, en lugar de definir de forma general «TCP permitido»;
- observar SYN backlog, SYN cookies y tasas de conexión antes de que límites de tasa estrictos afecten a servidores de correo legítimos;
- aceptar Proxy Protocol únicamente entre pares explícitamente fiables;
- documentar la terminación TCP y TLS para que la IP de origen, el timeout y la responsabilidad del RST sigan siendo trazables;
- proteger las capturas de paquetes y los diagnósticos de socket como datos operativos potencialmente sensibles;
- evitar manipulaciones innecesarias de middleboxes en MSS, opciones y temporizadores de inactividad, y probar cada desviación necesaria.

## Historia técnica

La separación hoy evidente entre IP y TCP surgió solo durante las primeras etapas del desarrollo de Internet. [RFC 675](https://datatracker.ietf.org/doc/html/rfc675) describía en 1974 todavía un Internet Transmission Control Program conjunto. La posterior división asignó a IP el transporte de paquetes entre redes y a TCP la sesión fiable de extremo a extremo. [RFC 793](https://datatracker.ietf.org/doc/html/rfc793) estandarizó este TCP en 1981.

| Año | Desarrollo |
|---:|---|
| 1974 | RFC 675 describe el Internet Transmission Control Program como precursor |
| 1981 | RFC 793 define TCP; RFC 791 describe el Internet Protocol separado |
| 1989 | RFC 1122 consolida los requisitos de host y precisa numerosas reglas TCP |
| 1992 | RFC 1323 introduce ventanas grandes y Timestamps para enlaces de alto rendimiento |
| 2009 | RFC 5681 consolida Slow Start, Congestion Avoidance, Fast Retransmit y Fast Recovery |
| 2011 | RFC 6298 estandariza el cálculo del Retransmission Timeout |
| 2014 | RFC 7323 sustituye RFC 1323 y actualiza Window Scale y Timestamps |
| 2022 | RFC 9293 sustituye RFC 793 y vuelve a reunir el estándar básico distribuido durante décadas |

Las implementaciones siguieron evolucionando en paralelo: Selective Acknowledgment, Explicit Congestion Notification, algoritmos modernos de control de congestión, TCP Fast Open y el offloading del sistema operativo amplían el núcleo. Sin embargo, para la búsqueda de errores sigue siendo válido el modelo original de par de sockets, espacio de secuencia, ventanas, estados y temporizadores.

## Fuentes

- [RFC Editor – RFC 9293: Transmission Control Protocol](https://www.rfc-editor.org/info/rfc9293)
- [RFC 9293 – Transmission Control Protocol](https://datatracker.ietf.org/doc/html/rfc9293) – especificación TCP consolidada, cabecera, estados, modelo de secuencia, interfaces y seguridad.
- [IANA – Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml) – servicios y rangos de puertos registrados.
- [RFC 6298 – Computing TCP's Retransmission Timer](https://datatracker.ietf.org/doc/html/rfc6298) – algoritmo de RTT, RTO y backoff.
- [RFC 5681 – TCP Congestion Control](https://datatracker.ietf.org/doc/html/rfc5681) – Slow Start, Congestion Avoidance, Fast Retransmit y Fast Recovery.
- [Wireshark User’s Guide – TCP Analysis](https://www.wireshark.org/docs/wsug_html_chunked/ChAdvTCPAnalysis.html) – derivación de retransmisión, Zero Window, Out-of-Order y otras señales de análisis.
- [RFC 7323 – TCP Extensions for High Performance](https://datatracker.ietf.org/doc/html/rfc7323) – Window Scale, Timestamps, RTTM y PAWS.
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) – diagnóstico de puertos TCP, resolución de nombres y rutas en Windows.
- [OpenBSD Manual – nc(1)](https://man.openbsd.org/nc.1) – conexiones TCP/UDP, listeners y pruebas de puertos con Netcat.
- [OpenBSD Manual – telnet(1)](https://man.openbsd.org/telnet.1) – cliente Telnet, establecimiento de conexiones y comportamiento del protocolo.
- [iproute2 – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html) – estados de socket, temporizadores, colas y filtros.
- [Microsoft Learn – Pktmon](https://learn.microsoft.com/windows-server/networking/technologies/pktmon/pktmon) – monitor de paquetes de Windows y conversión PCAPNG.
- [tcpdump(1) Manual](https://www.tcpdump.org/manpages/tcpdump.1.html) – opciones de captura, filtros y archivos de salida.
- [RFC 675 – Specification of Internet Transmission Control Program](https://datatracker.ietf.org/doc/html/rfc675) – precursor técnico de TCP e IP.
- [RFC 793 – Transmission Control Protocol](https://datatracker.ietf.org/doc/html/rfc793) – especificación TCP original de 1981.
