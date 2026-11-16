---
title: "Resolución de problemas: evidencia, dependencias y diagnóstico controlado"
blatt: "troubleshooting"
description: "Diagnóstico sistemático para administradores de mensajería e infraestructura: alcance, cronología de evidencias, grafos de dependencias, estados de protocolos, registros, métricas, trazas, captura de paquetes, pruebas de hipótesis, estabilización y resolución de causas."
fakten:
  - label: Método básico
    wert: Observación + modelo del sistema → hipótesis → prueba diferenciadora → hallazgo
    href: https://sre.google/sre-book/effective-troubleshooting/
  - label: Primer conjunto de datos
    wert: esperado · real · reproducible · ventana temporal · alcance · impacto
    href: https://sre.google/sre-book/effective-troubleshooting/
  - label: Tres perspectivas de diagnóstico
    wert: grafo de dependencias · estado del protocolo · cronología común
    href: https://opentelemetry.io/docs/concepts/signals/
  - label: Clases de evidencia
    wert: métricas · registros · trazas · consultas de estado · captura de paquetes
    href: https://sre.google/workbook/monitoring/
  - label: Correlación importante
    wert: Message-ID, Queue-ID, Trace-ID, Session-ID y host siempre con marca de tiempo
    href: https://www.w3.org/TR/trace-context/
  - label: Triaje
    wert: limitar el impacto y preservar la evidencia; el análisis de causas puede seguir después
    href: https://sre.google/sre-book/effective-troubleshooting/
  - label: Límite de aceptación de correo
    wert: 250 tras End-of-Data transfiere la responsabilidad SMTP al receptor
    href: https://datatracker.ietf.org/doc/html/rfc5321
  - label: Clases de errores SMTP
    wert: 4yz temporal · 5yz permanente; Enhanced Status Code proporciona la subclase
    href: https://datatracker.ietf.org/doc/html/rfc3463
  - label: Pruebas activas
    wert: ejecutar desde el origen afectado, con la misma identidad, selección de destino y política
    href: https://sre.google/sre-book/effective-troubleshooting/
  - label: Tiempo
    wert: el estado de NTP, la zona horaria y el desfase del reloj deben figurar en todo hallazgo distribuido
    href: https://datatracker.ietf.org/doc/html/rfc5905
  - label: Hallazgo negativo
    wert: documenta qué hipótesis se refutó en qué condiciones
    href: https://sre.google/sre-book/effective-troubleshooting/
  - label: Cierre
    wert: verificar servicio, ruta de datos y monitorización; documentar por separado causa y prevención
    href: https://csrc.nist.gov/pubs/sp/800/61/r3/final
werbung:
  - tools
  - newsletter
ctaThemen:
  - troubleshooting
  - messaging
  - betrieb
translationSourceHash: 126551190a29508cedf6aa90a1863b0b8a005ef6d04862bac0feb8d9dca5a769
translationModel: gpt-5.6-terra
translatedAt: 2026-11-13T03:26:15.155Z
translationReview: automatic
---

# Resolución de problemas: evidencia, dependencias y diagnóstico controlado

La resolución de problemas es la obtención controlada de conocimiento sobre un sistema defectuoso. El punto de partida no es la causa favorita más probable, sino una desviación precisa entre el comportamiento esperado y el observado. A partir de observaciones y un modelo técnico surgen hipótesis; una prueba debe posteriormente distinguir al menos entre dos causas posibles. Google SRE describe este procedimiento como la aplicación del método hipotético-deductivo y subraya que una búsqueda de fallos exitosa requiere tanto una metodología general como conocimiento de la arquitectura concreta del sistema ([Google SRE – Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)).

Para sistemas de mensajería distribuidos, la fórmula «comprobar de abajo arriba» no basta. Un mensaje puede haber sido aceptado mediante DNS, TCP y TLS funcionales y aun así quedar bloqueado en una cola, un motor de políticas, una inspección de contenido, una base de datos o en el siguiente salto. A la inversa, un error de usuario en una interfaz web puede deberse a la identidad, el estado de sesión, la API, el almacenamiento del buzón o la búsqueda. El diagnóstico se vuelve sólido cuando el administrador combina tres modelos: **el grafo de dependencias de la operación afectada, los autómatas de estado de los protocolos utilizados y una cronología alineada entre todos los componentes**.

La explicación comienza con el síntoma observable y lleva, mediante el alcance, la cronología y las evidencias, a una hipótesis comprobable. El cambio, la verificación y el seguimiento solo se realizan cuando causa y efecto se mantienen diferenciados.

La resolución de problemas comienza con una desviación observable: ¿quién está afectado, desde cuándo, en qué ruta y con qué resultado? Solo a partir de este alcance surge una hipótesis comprobable. El artículo abarca desde la delimitación, la evidencia y las pruebas hasta la resolución verificada.

## Modelo de diagnóstico: alcance, estado y evidencia

Un buen informe de problemas contiene el comportamiento esperado, el comportamiento real y, en la medida de lo posible, una ejecución reproducible. Antes incluso del análisis técnico se determina el alcance: quién o qué está afectado, desde cuándo, en qué dirección, a través de qué punto de entrada y con qué impacto. Este triaje decide si primero debe investigarse un caso individual, redirigirse el tráfico, revertirse un cambio defectuoso o abrirse un incidente mayor. Ante una interrupción grave, la estabilización tiene prioridad sobre el análisis completo de la causa; al mismo tiempo, deben conservarse los registros volátiles y otras evidencias ([Google SRE – Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/), [NIST SP 800-61 Rev. 3](https://csrc.nist.gov/pubs/sp/800/61/r3/final)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-troubleshooting.svg?v=20260813" title="Interaktive Infografik: evidenzbasierter Diagnosefluss vom Problembericht über Scope, Zeitachse, Abhängigkeiten, Protokollzustände und Signale bis zum kontrollierten Hypothesentest und verifizierten Abschluss" loading="lazy">
  <a href="/images/kb-interaktiv-troubleshooting.svg?v=20260813">Abrir directamente el gráfico interactivo</a>.
</iframe>

### Matriz de alcance para el primer hallazgo

La primera nota del administrador no debería decir «el correo no funciona», sino describir una transacción reproducible. Para mensajería e infraestructura son útiles al menos los siguientes campos:

| Campo | Contenido sólido | Por qué reduce el espacio de error |
|---|---|---|
| Esperado / real | «El destino SMTP debe devolver 250 después de DATA; se observa 451 4.4.1» | separa la expectativa funcional del síntoma |
| Conjunto afectado | usuario individual, dominio remitente, tenant, ubicación, protocolo o todas las transacciones | distingue errores de objeto, política, ruta y plataforma |
| Dirección y entrada | entrante/saliente, Relay/Submission/Access, GUI/API | identifica listener, política y límite de responsabilidad |
| Identidad y autorización | usuario, Service Principal, certificado, IP de origen, rol | permite comprobar autenticación y autorización |
| Periodo exacto | primer/último error en UTC, frecuencia, duración, zona horaria | alinea registros, métricas, trazas y cambios |
| Valores de correlación | Message-ID, Queue-ID, Trace-ID, Session-ID, host y destino | vincula eventos entre componentes |
| Reproducibilidad | determinista/intermitente, entrada concreta, caso de comparación funcional | permite pruebas diferenciadoras |
| Impacto y solución temporal | perdido, retrasado, duplicado, solo visualización; alternativa y riesgo | guía el triaje y la prioridad |
| Últimos cambios | despliegue, configuración, certificado, DNS, firewall, migración de datos, perfil de carga | proporciona hipótesis, pero aún no una prueba causal |

La formulación de cada entrada separa **observado**, **inferido** y **desconocido**. «La conexión TCP finaliza tras 21 segundos con un tiempo de espera agotado» es observado. «El firewall bloquea» es una hipótesis mientras no la respalde un contador de reglas, un hallazgo de paquetes o un rechazo administrativo. Esta disciplina lingüística evita que las suposiciones se conviertan inadvertidamente en hechos.

## Grafo de dependencias en lugar de una escalera de capas universal

OSI o TCP/IP explican la encapsulación de protocolos, pero solo representan parcialmente las dependencias operativas. Para la operación concreta, un administrador necesita un grafo dirigido: qué componente selecciona el destino, cuál abre la conexión, cuál comprueba la identidad y la política, dónde se asume el estado de forma persistente y qué servicios posteriores pueden impedir la finalización. Google SRE recomienda investigar entradas y salidas conocidas en límites de componentes bien definidos y acotar sistemas grandes linealmente o por bisección ([Google SRE – Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)).

Para un flujo de correo saliente, la ruta puede ser así:

1. La aplicación remitente genera el sobre y el contenido del mensaje y los entrega a un punto de conexión de Submission o Relay.
2. [DNS](/kb/dns) proporciona datos MX y de direcciones; el resolvedor, la caché, los sufijos de búsqueda y el DNS dividido pueden generar respuestas diferentes.
3. [TCP](/kb/tcp) conecta una dirección de origen concreta con una dirección y un puerto de destino concretos.
4. [TLS](/kb/tls) negocia parámetros y comprueba, según la política de transporte, el nombre, la cadena de certificados u otras anclas de confianza.
5. [SMTP](/kb/smtp) recorre Greeting, EHLO, autenticación opcional, sobre y DATA. Una respuesta `250` tras el punto final transfiere la responsabilidad al servidor SMTP receptor ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).
6. El receptor escribe el mensaje en la cola o el almacén, aplica reglas de enrutamiento, malware, DLP, archivado y cifrado, e intenta el siguiente salto.
7. El servicio de buzón, el índice, el acceso del cliente y la notificación determinan si el mensaje aceptado resulta visible para el usuario.

El diagnóstico pregunta en cada arista por una **transición comprobable**. Una conexión TCP exitosa solo demuestra el handshake TCP con el destino probado. No demuestra ni la confianza TLS ni la aceptación SMTP. Una respuesta SMTP `250` después de End-of-Data demuestra la asunción por este salto, pero no el almacenamiento en el buzón final. Una salida exitosa de la cola no demuestra que un índice de búsqueda ya muestre el mensaje.

Además de la ruta directa, el grafo debe incluir dependencias transversales: hora, resolución de nombres, identidad, certificados y claves, configuración, base de datos, almacenamiento de objetos o archivos, cola, balanceador de carga, proxy, firewall, observabilidad y proveedores externos. Un error en una dependencia de este tipo puede afectar simultáneamente a varios componentes aparentemente independientes.

## Estados de protocolo y límites de responsabilidad

Los protocolos ofrecen más valor diagnóstico que la mera afirmación «puerto abierto». El hallazgo relevante es **qué estado se alcanzó, qué parte envió por última vez un mensaje válido y qué respuesta o tiempo de espera agotado siguió**.

- **DNS:** El nombre de consulta, tipo, resolvedor consultado, RCODE, Answer/Authority, TTL y estado DNSSEC deben considerarse conjuntamente. El almacenamiento en caché forma parte del diseño de DNS; por ello, distintos resolvedores pueden tener temporalmente diferentes estados de datos ([RFC 1034](https://datatracker.ietf.org/doc/html/rfc1034)).
- **TCP:** La dirección y puerto de origen/destino, el estado de conexión, las retransmisiones y el cierre distinguen entre listener ausente, descarte por política, rechazo activo y interrupción tras un establecimiento exitoso. RFC 9293 especifica estados y procesamiento de segmentos; un eco ICMP no sustituye al servicio TCP probado ([RFC 9293](https://datatracker.ietf.org/doc/html/rfc9293), [RFC 4443](https://datatracker.ietf.org/doc/html/rfc4443)).
- **TLS:** ClientHello, ServerHello, cadena de certificados, nombre, almacén de confianza, protocolo negociado y alerta localizan el error. TLS 1.3 define las alertas como contenido de protocolo propio; por tanto, «Handshake failure» es un estado, no simplemente un sinónimo de error de red ([RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446)).
- **SMTP:** Cada respuesta pertenece a un comando y una fase de sesión. `4yz` significa una respuesta negativa temporal, `5yz` una permanente; los Enhanced Status Codes dividen el estado en clase, asunto y detalle ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321), [RFC 3463](https://datatracker.ietf.org/doc/html/rfc3463)).
- **LDAP:** Un establecimiento TCP/TLS exitoso no dice nada sobre Bind, Search Base, Filter, Controls ni derechos de acceso. LDAPResult transmite `resultCode`, `matchedDN`, mensaje de diagnóstico y Referrals opcionales ([RFC 4511](https://datatracker.ietf.org/doc/html/rfc4511)).
- **HTTP:** El código de estado, método, URI de destino, saltos de proxy/gateway, encabezados y cuerpo de respuesta constituyen un hallazgo. Un `403` no significa lo mismo que un establecimiento de conexión fallido; RFC 9110 define la semántica de los códigos de estado ([RFC 9110](https://datatracker.ietf.org/doc/html/rfc9110)).
- **IMAP:** Las respuestas etiquetadas completan un comando; los datos sin etiqueta pueden informar cambios de estado independientemente. Estados como not authenticated, authenticated y selected limitan qué comandos están permitidos ([RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051)).

Así, la pregunta pasa de «¿funciona el servidor X?» a «¿qué último estado confirmado alcanza la transacción Y con estas entradas en esta ruta?».

Cuando se conocen las dependencias y la fase de protocolo, se eligen las señales adecuadas. Las métricas muestran alcance y momento, los registros muestran decisiones, las trazas muestran la ruta de llamada y los datos de paquetes muestran el diálogo de red real.

## Señales: métricas, registros, trazas y datos de paquetes

Ninguna señal individual representa la verdad de todo el sistema. Las métricas condensan muchos eventos en series temporales, los registros describen eventos discretos, las trazas vinculan operaciones a lo largo de una solicitud, las consultas de estado muestran el estado actual y una captura de paquetes documenta el tráfico de red observado en un punto de medición. Google SRE menciona monitorización, registros y trazado como medios de diagnóstico centrales; OpenTelemetry distingue, entre otras, traces, metrics y logs como señales de telemetría ([Google SRE – Monitoring](https://sre.google/workbook/monitoring/), [OpenTelemetry – Signals](https://opentelemetry.io/docs/concepts/signals/)).

| Señal | Responde especialmente bien | Límite típico |
|---|---|---|
| Métrica | ¿Cuándo comenzó la desviación, cuál es su magnitud, qué población está afectada? | La agregación oculta casos individuales; las etiquetas de alta cardinalidad son costosas |
| Registro estructurado | ¿Qué evento ocurrió en un componente y con qué campos? | La falta de contexto, el muestreo, la rotación o un reloj incorrecto dificultan la correlación |
| Traza | ¿Qué saltos y duraciones tuvo una solicitud individual? | Solo funciona a través de límites instrumentados que propaguen contexto |
| Consulta de estado | ¿Qué se aplica ahora al listener, la cola, la replicación, el certificado o la configuración? | Puede diferir del momento del error y modificar el estado |
| Captura de paquetes | ¿Qué paquetes vio este punto de medición de interfaz/host? | No ofrece visibilidad de la aplicación interna, del contenido cifrado ni de otra ruta |

El estándar W3C Trace Context define `traceparent` y `tracestate`, para que la identidad de trazado pueda propagarse entre componentes HTTP. No es una ID de correo universal. Por ello, en una plataforma de mensajería deben vincularse explícitamente la Trace-ID, la RFC-Message-ID, las Queue-ID de cada MTA, la Session-ID y el host de destino ([W3C Trace Context](https://www.w3.org/TR/trace-context/)). Syslog estandariza un formato de eventos con datos estructurados, pero por sí solo no garantiza ni la integridad ni la conservación centralizada; la gestión de registros incluye generación, transmisión, almacenamiento, acceso y eliminación ([RFC 5424](https://datatracker.ietf.org/doc/html/rfc5424), [NIST SP 800-92](https://csrc.nist.gov/pubs/sp/800/92/final)).

### La cronología común

Una cronología no solo incluye el supuesto momento del error, sino al menos el primer y último éxito conocidos, el primer y último error, despliegues y cambios de configuración relevantes, cambios de certificados o tokens, failovers, reinicios y la base temporal de cada evidencia. NTP Version 4 sirve para sincronizar los relojes de los sistemas; aun así, los administradores deben comprobar el desfase, Stratum/fuente, estado Leap/Sync, zona horaria y posibles saltos manuales ([RFC 5905](https://datatracker.ietf.org/doc/html/rfc5905)).

«Marca de tiempo 10:14» no es un punto de referencia común sin zona horaria, host y estado del reloj. En colas asíncronas, además, el momento del evento de recepción, procesamiento y exportación del registro puede diferir. Una nota sólida conserva la marca de tiempo original y añade una hora UTC normalizada, en lugar de sobrescribir el valor original.

Las señales por sí solas no explican la causa. Una buena hipótesis predice qué resultado debe producirse en una prueba específica y cómo se distingue de una alternativa.

## Hipótesis y pruebas diferenciadoras

Una hipótesis es útil si una prueba de bajo riesgo puede refutarla o hacerla más probable frente a una alternativa. «Problema de red» es demasiado amplio. «El MTA no puede alcanzar por TCP 25 la dirección IPv6 elegida por su resolvedor, mientras que IPv4 funciona» indica origen, selección de destino, protocolo y comparación.

Un plan de pruebas documenta antes de la ejecución:

1. qué dos o más hipótesis separa la prueba;
2. desde qué host, namespace, interfaz y usuario se ejecuta;
3. qué configuración de DNS, proxy, enrutamiento, identidad y confianza utiliza realmente;
4. qué resultado se espera en cada caso;
5. qué efectos secundarios, accesos a datos o carga genera;
6. cómo se revierte un cambio de prueba;
7. qué datos sin procesar y marcas de tiempo se conservan.

Google SRE señala que las pruebas pueden inducir a error debido a orígenes distintos, políticas de firewall, credenciales u otros factores perturbadores. La propia prueba también puede afectar al sistema: el registro detallado modifica la carga y el timing; un reinicio elimina el estado volátil; un reintento puede provocar un mensaje duplicado. Por ello, las pruebas deben ejecutarse desde el origen afectado y, en lo posible, con la misma selección de destino, identidad y política que el proceso defectuoso ([Google SRE – Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)).

La comparación con un caso funcional es especialmente sólida si difiere exactamente una variable independiente: el mismo usuario en otro host, otro usuario en el mismo host, el mismo mensaje a otro dominio o el mismo host de destino mediante otra familia de direcciones. En cambio, varios cambios de configuración simultáneos crean un estado de sistema nuevo y desconocido.

## Herramientas de administración a lo largo de la ruta real

Los siguientes comandos son ejemplos de diagnóstico, no comprobaciones universales de estado. Los marcadores de posición deben adaptarse al origen afectado, resolvedor, host, puerto y periodo. Las credenciales, los tokens y el contenido de los mensajes no deben incluirse sin censurar en tickets ni en el historial de shell.

### Registrar la respuesta DNS y el resolvedor elegido

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName example.ch -Type MX -DnsOnly -Server 192.0.2.53
Resolve-DnsName mx1.example.ch -Type A -DnsOnly -Server 192.0.2.53
Resolve-DnsName mx1.example.ch -Type AAAA -DnsOnly -Server 192.0.2.53</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig @192.0.2.53 +noall +answer +authority example.ch MX
dig @192.0.2.53 +noall +answer mx1.example.ch A
dig @192.0.2.53 +noall +answer mx1.example.ch AAAA</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) y [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) hacen explícitos el resolvedor y el tipo de registro. La comparación con un servidor público o autoritativo puede revelar diferencias de DNS dividido y caché, pero no debe confundirse con el resolvedor que utiliza realmente el servicio.

### Comprobar la conexión TCP y la ruta desde el origen afectado

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für TCP- und Pfaddiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection mx1.example.ch -Port 25 -InformationLevel Detailed
tracert -d mx1.example.ch</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 mx1.example.ch 25
traceroute -n mx1.example.ch</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) prueban el establecimiento TCP; [`tracert`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/tracert) y [`traceroute`](https://man7.org/linux/man-pages/man8/traceroute.8.html) investigan límites de saltos y respuestas a lo largo de una ruta. Un asterisco en la ruta no demuestra una interrupción del tráfico útil, y una prueba de eco ICMP exitosa no demuestra un puerto TCP abierto. El histórico [`ping`](https://ftp.arl.army.mil/~mike/ping.html) sigue siendo útil como prueba de latencia/accesibilidad, pero tampoco es una prueba de aplicación.

### Observar TLS y el diálogo de aplicación

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für TLS- und Protokolldiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">openssl.exe s_client -connect mx1.example.ch:25 -starttls smtp -servername mx1.example.ch -showcerts
curl.exe --verbose --connect-timeout 5 https://api.example.ch/health</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">openssl s_client -connect mx1.example.ch:25 -starttls smtp -servername mx1.example.ch -showcerts
curl --verbose --connect-timeout 5 https://api.example.ch/health</code></pre>
  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) muestra el handshake TLS, los certificados y las alertas; [`curl`](https://curl.se/docs/manpage.html) puede hablar, entre otros, HTTP, SMTP, IMAP, POP3 y LDAP. Ambos utilizan su propia configuración de compilación y confianza, que puede diferir de la del producto afectado. [`telnet`](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/telnet) o la [página de manual de OpenBSD para `telnet`](https://man.openbsd.org/telnet.1) se usan históricamente para diálogos en texto plano, pero no proporcionan comprobación TLS; para pruebas TCP sencillas, `nc` suele ser más inequívoco.

### Capturar listeners locales y estados de conexión

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokale Socketdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-NetTCPConnection -LocalPort 25,443 -ErrorAction SilentlyContinue |
  Sort-Object LocalPort,State,RemoteAddress |
  Format-Table LocalAddress,LocalPort,RemoteAddress,RemotePort,State,OwningProcess</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ss -lntp '( sport = :25 or sport = :443 )'
ss -nto state syn-sent</code></pre>
  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) y [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) muestran puntos de conexión locales y estados TCP. También son decisivos la dirección de enlace, el namespace/contenedor, el proceso propietario y la familia de direcciones: un listener en `127.0.0.1` o en otro namespace de red no es automáticamente accesible desde fuera.

### Captura de paquetes con filtro y ventana temporal acotados

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Paketmitschnitt">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">pktmon filter remove
pktmon filter add Mailflow -t TCP SYN ACK RST FIN -p 25
pktmon start --capture --pkt-size 0 --file-name C:\\Temp\\mailflow.etl
pktmon stop
pktmon etl2pcap C:\\Temp\\mailflow.etl --out C:\\Temp\\mailflow.pcapng</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sudo tcpdump -i any -nn -s 0 -w /tmp/mailflow.pcap \
  'host 192.0.2.25 and tcp port 25'</code></pre>
  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) y [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) capturan paquetes en el punto de medición elegido. Antes deben definirse destino, filtro, duración, espacio de almacenamiento y protección de datos. Una captura puede contener credenciales, contenido personal y material de claves. Además, «no aparece en la captura» solo demuestra que el paquete no se vio en ese punto de medición con ese filtro.

### Conservar registros del periodo exacto del error

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Logdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$start = [datetime]'2026-08-08T10:00:00Z'
$end   = [datetime]'2026-08-08T10:15:00Z'
Get-WinEvent -FilterHashtable @{LogName='System'; StartTime=$start; EndTime=$end} |
  Select-Object TimeCreated,Id,ProviderName,LevelDisplayName,Message</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">journalctl --since '2026-08-08 10:00:00 UTC' \
  --until '2026-08-08 10:15:00 UTC' \
  --output short-iso-precise --no-pager</code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent) y [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) delimitan temporalmente la consulta. El hallazgo debe incluir Event-ID/nombres de campo y la hora sin procesar; un mensaje de error traducido libremente es peor para buscar que el código sin modificar. Antes de una rotación o un reinicio se exportan los registros relevantes y se documenta su procedencia.

### Comprobar el reloj y la fuente de sincronización

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Zeitdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">w32tm /query /status
w32tm /query /source
Get-Date -AsUTC -Format o</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">chronyc tracking
chronyc sources -v
date --utc --iso-8601=ns</code></pre>
  </div>
</div>

[`w32tm`](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings#w32tm-command-line-tool) y [`chronyc`](https://chrony-project.org/doc/4.7/chronyc.html) muestran la fuente y el estado de sincronización; la salida UTC local proporciona una referencia que puede documentarse inmediatamente. Un reloj sincronizado aún no garantiza que una aplicación escriba la misma marca de tiempo o la misma zona horaria.

En mensajería, este modelo se aplica a lo largo del mensaje: aceptación, política, cola, siguiente salto, buzón y acceso del cliente poseen cada uno sus propias evidencias.

## Diagnóstico de mensajería como cadena de responsabilidad

En un mensaje, la pregunta más importante no es «¿dónde falta?», sino **¿qué componente ha asumido demostrablemente su responsabilidad y a quién se la ha transferido?** Cada salto SMTP tiene su propia sesión y normalmente su propia Queue-ID. El encabezado RFC-Message-ID se transporta en el contenido del mensaje, pero no es necesariamente único ni fiable; las Queue-ID son locales y cambian en cada salto.

Por tanto, un seguimiento sólido documenta:

1. remitente y destinatario del sobre por separado de las direcciones visibles en los encabezados;
2. Message-ID, tamaño y una huella compatible con la protección de datos o asunto de prueba;
3. sesión de cliente/Submission y el primer hallazgo de aceptación `250`;
4. cada asignación de Queue-ID de entrada a salida;
5. decisión de política, cuarentena, reescritura, división o rebote;
6. siguiente salto seleccionado, respuesta DNS, IP de origen/destino, resultado TLS y SMTP;
7. aceptación final o error permanente para cada destinatario;
8. hallazgo de buzón, índice y cliente, si la aceptación de transporte fue exitosa.

Un SMTP-`451 4.4.1` no es lo mismo que un «5xx» genérico. El primer dígito del código SMTP y el Enhanced Status Code deben conservarse literalmente. Los errores temporales suelen conducir a cola y reintentos; los errores permanentes pueden desencadenar un NDR/DSN. Que un mensaje pueda reenviarse depende de si ya se ha superado un límite de aceptación. Repetir a ciegas tras una respuesta perdida puede generar duplicados ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321), [RFC 3463](https://datatracker.ietf.org/doc/html/rfc3463)).

## Separar triaje, resolución y análisis de causas

Cuando el impacto es alto, deben distinguirse tres líneas de trabajo:

- **Estabilizar:** redirigir tráfico, limitar la función defectuosa, proteger la capacidad, detener la corrupción de datos o ejecutar una reversión segura conocida.
- **Conservar evidencia:** asegurar registros, estado de la cola, configuración, métricas, trazas, datos de paquetes, estado temporal y referencias de cambios antes de que roten o desaparezcan tras un reinicio.
- **Diagnosticar:** priorizar hipótesis y refutarlas o respaldarlas con pruebas de riesgo mínimo.

Una resolución temporal no demuestra automáticamente la causa. Si un reinicio finaliza la interrupción, siguen siendo posibles fugas de memoria, deadlocks, pools de conexión agotados, caché dañada, timeout externo y muchas otras causas. El reinicio debe registrarse como intervención con momento y efecto; posteriormente se planifican prevención y evidencia adicional.

También «último cambio» es una fuente de hipótesis, no una prueba de causalidad. La correlación temporal puede surgir por un desencadenante común o por casualidad. Google SRE advierte expresamente contra síntomas irrelevantes, teorías favoritas improbables y correlaciones espurias ([Google SRE – Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)).

Un cambio solo finaliza el incidente cuando la ruta original vuelve a funcionar, se excluyen efectos secundarios y la monitorización confirma el estado estable.

## Verificación y criterios de cierre

Un incidente no se cierra cuando una única prueba tiene éxito. La aceptación comprueba la operación funcional original, el alcance afectado y las áreas de fallo adyacentes:

- transacciones representativas en ambas direcciones relevantes;
- acumulación y vaciado de la cola, tasa de reintentos y errores;
- consistencia de datos, replicación, búsqueda y vista del cliente;
- autenticación, autorización, rutas de certificados y claves;
- monitorización, alertas y paneles desde la perspectiva del usuario;
- nueva comprobación después de al menos un intervalo relevante de temporizador, reintento o caché;
- ninguna nueva brecha de datos, ningún duplicado inesperado y ninguna degradación oculta.

El informe de cierre separa síntoma, impacto, cronología, causa inmediata, factores contribuyentes, recuperación y medidas permanentes. Se conservan los hallazgos negativos: documentan qué hipótesis se refutaron en qué condiciones y evitan trabajo repetido. NIST SP 800-61 Rev. 3 integra la respuesta a incidentes en detectar, responder, recuperar y mejorar toda la organización ([NIST SP 800-61 Rev. 3](https://csrc.nist.gov/pubs/sp/800/61/r3/final)).

## Evolución técnica de las herramientas de diagnóstico

El diagnóstico de red temprano hizo directamente visibles propiedades individuales de protocolos. Mike Muuss escribió `ping` en 1983 para 4.2a BSD basándose en ICMP Echo; posteriormente, `traceroute` de Van Jacobson utilizó límites de saltos IP que expiran y respuestas ICMP para hacer visibles las estaciones de ruta ([Mike Muuss – The Story of the PING Program](https://ftp.arl.army.mil/~mike/ping.html)). Estas herramientas siguen siendo valiosas, pero solo responden preguntas definidas de forma estricta.

Con las aplicaciones distribuidas, la necesidad de diagnóstico pasó de la accesibilidad del host a la correlación de transacciones. Syslog obtuvo un formato de mensajes estandarizado, el trazado distribuido conectó operaciones individuales, W3C Trace Context estandarizó la propagación HTTP de identidad de trazado y OpenTelemetry reunió traces, metrics y logs en un marco de observabilidad independiente de proveedores ([RFC 5424](https://datatracker.ietf.org/doc/html/rfc5424), [W3C Trace Context](https://www.w3.org/TR/trace-context/), [OpenTelemetry – Signals](https://opentelemetry.io/docs/concepts/signals/)). El núcleo metodológico permaneció igual: una herramienta solo es tan buena como la pregunta precisa, el punto de medición y la interpretación documentada de su resultado.

## Fuentes

- [Google SRE – Effective Troubleshooting](https://sre.google/sre-book/effective-troubleshooting/)
- [NIST – SP 800-61 Rev. 3, Incident Response](https://csrc.nist.gov/pubs/sp/800/61/r3/final)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [IETF RFC 1034 – Domain Names: Concepts and Facilities](https://datatracker.ietf.org/doc/html/rfc1034)
- [IETF RFC 9293 – Transmission Control Protocol](https://datatracker.ietf.org/doc/html/rfc9293)
- [IETF RFC 4443 – ICMPv6](https://datatracker.ietf.org/doc/html/rfc4443)
- [IETF RFC 8446 – TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [IETF RFC 3463 – Enhanced Mail System Status Codes](https://datatracker.ietf.org/doc/html/rfc3463)
- [IETF RFC 4511 – LDAP Protocol](https://datatracker.ietf.org/doc/html/rfc4511)
- [IETF RFC 9110 – HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc9110)
- [IETF RFC 9051 – IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051)
- [Google SRE Workbook – Monitoring](https://sre.google/workbook/monitoring/)
- [OpenTelemetry – Signals](https://opentelemetry.io/docs/concepts/signals/)
- [W3C – Trace Context](https://www.w3.org/TR/trace-context/)
- [IETF RFC 5424 – The Syslog Protocol](https://datatracker.ietf.org/doc/html/rfc5424)
- [NIST – SP 800-92, Log Management](https://csrc.nist.gov/pubs/sp/800/92/final)
- [IETF RFC 5905 – Network Time Protocol Version 4](https://datatracker.ietf.org/doc/html/rfc5905)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Microsoft Learn – tracert](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/tracert)
- [Linux man-pages – traceroute](https://man7.org/linux/man-pages/man8/traceroute.8.html)
- [Mike Muuss – The Story of the PING Program](https://ftp.arl.army.mil/~mike/ping.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [Microsoft Learn – telnet](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/telnet)
- [OpenBSD – telnet](https://man.openbsd.org/telnet.1)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Microsoft Learn – Packet Monitor (pktmon)](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – manual page](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [Microsoft Learn – Windows Time Service tools](https://learn.microsoft.com/en-us/windows-server/networking/windows-time-service/windows-time-service-tools-and-settings)
- [chrony – chronyc](https://chrony-project.org/doc/4.7/chronyc.html)
