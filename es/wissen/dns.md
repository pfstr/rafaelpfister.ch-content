---
title: "DNS: resolución, delegación y operación"
blatt: "dns"
description: "El sistema de nombres de dominio desde la perspectiva de un administrador: espacio de nombres, zonas y delegación, resolución recursiva, RRsets, caché y TTL, respuestas negativas, UDP y TCP, EDNS, DNSSEC, replicación de zonas y diagnóstico para infraestructuras de mensajería."
fakten:
  - label: Nombre completo
    wert: Domain Name System
    href: https://datatracker.ietf.org/doc/html/rfc1034
  - label: Modelo básico
    wert: espacio de nombres jerárquico distribuido y conjunto de datos
    href: https://datatracker.ietf.org/doc/html/rfc9499#section-2
  - label: Estándares fundamentales
    wert: RFC 1034 · RFC 1035 · STD 13
    href: https://www.rfc-editor.org/info/std13
  - label: Claves de consulta
    wert: QNAME · QTYPE · QCLASS
    href: https://datatracker.ietf.org/doc/html/rfc1035#section-4.1.2
  - label: Unidad de datos
    wert: Resource Record Set (RRset)
    href: https://datatracker.ietf.org/doc/html/rfc9499#section-5
  - label: Roles de servidor
    wert: autoritativo · recursivo · reenviador
    href: https://datatracker.ietf.org/doc/html/rfc9499#section-6
  - label: Transporte
    wert: UDP y TCP · Puerto 53
    href: https://datatracker.ietf.org/doc/html/rfc7766
  - label: Extensiones
    wert: EDNS(0) mediante OPT
    href: https://datatracker.ietf.org/doc/html/rfc6891
  - label: Consistencia
    wert: cachés positivas y negativas controladas por TTL
    href: https://datatracker.ietf.org/doc/html/rfc2308
  - label: Integridad
    wert: "DNSSEC: DNSKEY · DS · RRSIG · NSEC"
    href: https://datatracker.ietf.org/doc/html/rfc4034
  - label: Sincronización de zonas
    wert: NOTIFY · AXFR · IXFR
    href: https://datatracker.ietf.org/doc/html/rfc1996
  - label: Relación con el correo
    wert: MX · PTR · TXT y nombres de destino derivados
    href: https://datatracker.ietf.org/doc/html/rfc5321#section-5
werbung:
  - tools
  - newsletter
ctaThemen:
  - smtp-mailflow
translationSourceHash: f68d5b35d0c022538eb216baafcdf1c277fffbe2c2db0ed4a3b519c01ba63062
translationModel: gpt-5.6-terra
translatedAt: 2026-10-04T10:37:29.310Z
translationReview: required
---

# DNS: resolución, delegación y operación

DNS responde a la pregunta de qué información se publica para un nombre concreto y quién es responsable de ella. El sistema es a la vez un espacio de nombres jerárquico, una base de datos distribuida y un protocolo binario de consulta. No solo proporciona direcciones IP, sino también servidores de nombres, destinos de correo, puntos de conexión de servicios, claves y políticas. Por ello, quien considere DNS únicamente como «resolución de nombres» pasa por alto precisamente los registros de los que dependen la mensajería y la identidad ([RFC 9499](https://datatracker.ietf.org/doc/html/rfc9499)).

El siguiente recorrido comienza en el Stub Resolver de una aplicación, sigue la caché y las delegaciones hasta el servidor autoritativo y devuelve después la respuesta. Este proceso permite situar TTL, Glue, fallback a TCP, DNSSEC, operación de zonas y los patrones de error habituales.

Para los administradores de mensajería, DNS es un sistema de control previo. Un MTA determina mediante él el siguiente destino de correo, la autenticación del remitente lee políticas y claves, los procedimientos de certificados pueden incluir datos protegidos por DNSSEC, y los clientes de directorio o Kerberos localizan servicios mediante registros SRV. Sin embargo, DNS no comprueba si el servicio encontrado está en buen estado. Una respuesta MX sintácticamente correcta puede apuntar a un listener SMTP inalcanzable; una búsqueda A correcta no dice nada sobre TLS, autenticación o el estado de la aplicación.

## Arquitectura y roles

La responsabilidad sobre los nombres está organizada como un árbol. En la raíz sin nombre `.` comienzan dominios de nivel superior como `ch.`, seguidos por dominios delegados y otras etiquetas. Un punto final hace que un nombre sea completo e impide los sufijos de búsqueda locales. Esta pequeña diferencia de notación es relevante operativamente: `mail.example.ch` puede complementarse con un Search Domain en un cliente, mientras que `mail.example.ch.` no.

Un **dominio** es una parte del espacio de nombres. En cambio, una **zona** es el conjunto de datos administrativamente coherente del que un servidor autoritativo tiene responsabilidad local. Una delegación separa una zona hija de su zona padre. Para ello, el padre publica un RRset NS y, si el nombre de un servidor de nombres está dentro de la zona hija delegada, los datos A o AAAA necesarios para alcanzarlo como **Glue**. La distinción fundamental entre espacio de nombres, zonas y delegación procede de [RFC 1034](https://datatracker.ietf.org/doc/html/rfc1034#section-4.2); [RFC 9499, sección 7](https://datatracker.ietf.org/doc/html/rfc9499#section-7) resume la terminología actual.

Cuatro roles lógicos explican el proceso de resolución:

| Rol | Conocimiento y función | Límite operativo importante |
|---|---|---|
| Stub Resolver | recibe la consulta de la aplicación y la reenvía a un resolver configurado | los sufijos de búsqueda, el archivo hosts local y la caché del cliente pueden influir en el resultado antes del DNS propiamente dicho |
| Resolver recursivo | entrega una respuesta final desde la caché o mediante consultas iterativas | es el límite de confianza para caché, filtrado, registro y validación DNSSEC |
| Reenviador | asume consultas recursivas de otro resolver | desplaza la resolución y la observabilidad a otro operador |
| Servidor autoritativo | responde desde zonas cargadas localmente y establece el bit AA en respuestas autoritativas | no conoce la salud del servicio y no debería ofrecer recursión abierta para nombres ajenos |

Un producto puede implementar varios roles, pero operativamente deben considerarse por separado. Un error en el servicio autoritativo afecta a la publicación de las propias zonas; un error en el servicio recursivo afecta a la resolución de nombres de los propios clientes. Los procesos, direcciones o dominios de fallo compartidos dificultan esta distinción.

## Resolución de nombres paso a paso

Normalmente, una aplicación no consulta por sí misma servidores raíz y autoritativos. Su Stub Resolver entrega una consulta recursiva a un resolver. Si no hay una entrada útil en caché, este sigue las delegaciones desde la raíz, pasando por el dominio de nivel superior, hasta la zona responsable. Cada referral indica el siguiente RRset NS y, cuando es necesario, direcciones Glue. El resolver compone la respuesta final, valida DNSSEC si procede y almacena el resultado en caché ([RFC 1034, sección 4.3](https://datatracker.ietf.org/doc/html/rfc1034#section-4.3)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1016" src="/images/kb-interaktiv-dns-aufloesung.svg?v=20260813" title="Interaktive Infografik: DNS-Auflösung von Stub Resolver über Cache, Root und Delegationen bis zur autoritativen Antwort" loading="lazy">
  <a href="/images/kb-interaktiv-dns-aufloesung.svg?v=20260813">Abrir infografía sobre la resolución DNS</a>
</iframe>

El recorrido lineal dibujado es un modelo de arranque en frío. En una caché de resolver caliente, las delegaciones de raíz y TLD suelen estar ya presentes, por lo que solo es necesaria una parte de los pasos. La minimización de QNAME, el reenvío, las zonas locales o las cachés DNSSEC agresivas también pueden modificar la ruta de paquetes visible. Sigue siendo decisivo asignar cada observación a un rol: una respuesta de caché no autoritativa no prueba qué entrega en ese momento el servidor autoritativo responsable.

## Estructura técnica de un mensaje DNS

En la red, un mensaje DNS consta de Header, Question, Answer, Authority y Additional Section. La pregunta indica QNAME, QTYPE y QCLASS. Las respuestas y referencias aparecen como Resource Records en las demás secciones. Los flags indican, entre otras cosas, autoridad, solicitud de recursión y truncamiento; el Response Code describe el resultado. Por tanto, para el análisis no solo importa el texto de Answer, sino también de qué servidor procede y con qué flags llegó ([RFC 1035, sección 4.1](https://datatracker.ietf.org/doc/html/rfc1035#section-4.1)).

Para el diagnóstico son especialmente relevantes:

| Señal | Significado | Pregunta administrativa habitual |
|---|---|---|
| `AA` | La respuesta es autoritativa para el nombre respondido | ¿Se consultó directamente la zona responsable o solo una caché? |
| `TC` | La respuesta se truncó para el transporte utilizado | ¿Funciona el reintento por TCP y el firewall permite TCP/53? |
| `RD` / `RA` | Recursión solicitada / ofrecida por el servidor | ¿Se utilizó por error un servidor autoritativo como resolver? |
| `AD` | El validador que responde considera autenticados los datos | ¿Es fiable el resolver y está protegido el transporte hacia él? |
| `CD` | El cliente solicita que el resolver no descarte errores de validación | ¿Se está validando DNSSEC o solo examinando datos sin procesar? |
| `RCODE` | Estado del resultado como NOERROR, NXDOMAIN, SERVFAIL o REFUSED | ¿El nombre es incorrecto, el tipo no existe, la resolución está interrumpida o la consulta fue rechazada por una política? |

EDNS(0) añade flags, opciones y una carga útil UDP anunciada mayor mediante un **registro OPT** de tipo pseudo, sin sustituir el formato básico ([RFC 6891](https://datatracker.ietf.org/doc/html/rfc6891)). El bit DNSSEC DO se encuentra en este campo de flags ampliado, no en el Header DNS original.

## Resource Records y RRsets

El contenido técnico reside en los Resource Records. El Owner Name, el tipo y la clase determinan a qué pertenecen los datos; TTL y RDATA proporcionan el tiempo de caché y el valor específico del tipo. Todos los registros con el mismo propietario, tipo y clase forman un RRset y comparten un TTL. Por tanto, varios valores MX, A o AAAA son un conjunto almacenado conjuntamente en caché, no objetos individuales controlables de forma independiente ([RFC 2181, sección 5](https://datatracker.ietf.org/doc/html/rfc2181#section-5)).

| Tipo | Función | Límite importante |
|---|---|---|
| `SOA` | metadatos de zona, Serial, Refresh/Retry/Expire y parámetro de caché negativa | un RR SOA por zona en el Apex; el cambio de Serial controla la sincronización con los secundarios |
| `NS` | servidores autoritativos de una zona o delegación | el destino de un NS no puede ser un alias |
| `A` / [`AAAA`](https://datatracker.ietf.org/doc/html/rfc3596#section-2.1) | dirección IPv4 o IPv6 de un nombre | no indica nada sobre el puerto del servicio ni su accesibilidad |
| `CNAME` | alias de un nombre a un nombre canónico | en principio no puede coexistir con otros datos en el mismo propietario |
| `MX` | Mail Exchanger con Preference | el destino debe resolverse a A/AAAA y no puede ser un CNAME |
| `PTR` | resolución inversa, normalmente bajo `in-addr.arpa.` o `ip6.arpa.` | la zona inversa suele pertenecer al titular de la dirección, no al operador de la zona directa |
| `TXT` | una o más cadenas de caracteres sin semántica entre protocolos | la interpretación surge solo mediante SPF, DKIM, DMARC u otro procedimiento |
| [`SRV`](https://datatracker.ietf.org/doc/html/rfc2782) | servicio, transporte, prioridad, peso, puerto y destino | el cliente debe implementar la semántica SRV del servicio correspondiente |
| [`CAA`](https://datatracker.ietf.org/doc/html/rfc8659) | política de autoridades de certificación para nombres de dominio | no es cifrado de transporte ni un certificado de servidor |
| `DS`, `DNSKEY`, `RRSIG`, `NSEC` | cadena de confianza DNSSEC, claves, firmas y inexistencia autenticada | protege la integridad de los datos DNS, no su confidencialidad |

El texto de un archivo de zona es solo la **forma de presentación**. En la red, los nombres se transmiten por etiquetas, los números en binario y determinados nombres pueden comprimirse opcionalmente. Por ello, quien copie una cadena desde una interfaz de usuario debe distinguir si la interfaz ya ha compuesto comillas, secuencias de escape o varias cadenas TXT en una carga lógica.

## Delegación, Glue y autoridad

Con una delegación, una zona entrega la responsabilidad de un subárbol. El padre publica el RRset NS de la zona hija; la zona hija vuelve a indicar por sí misma sus servidores autoritativos. Si ambas partes difieren, los resolvers pueden seguir rutas distintas. Las direcciones Glue del padre solo resuelven el problema de la gallina y el huevo de la accesibilidad. Los valores A/AAAA autoritativos en el nombre del servidor de nombres siguen siendo datos independientes con su propio TTL y mantenimiento.

El **Glue in-bailiwick** es especialmente crítico: si `example.ch.` se delega en `ns1.example.ch.`, el resolver necesita la dirección de `ns1.example.ch.` antes de poder consultar la zona hija. Sin Glue se produciría un ciclo de resolución. En cambio, si la delegación apunta a `ns1.provider.net.`, su dirección puede determinarse mediante otra cadena de delegaciones.

Por tanto, al cambiar servidores de nombres deben comprobarse al menos cuatro estados: la nueva zona está cargada en todos los servidores, el RRset NS hijo está adaptado, la delegación padre está adaptada y el Glue necesario está actualizado. Solo entonces deben retirarse del servicio o de la accesibilidad los servidores antiguos.

## Caché, TTL y respuestas negativas

Tras la resolución, una respuesta continúa existiendo en las cachés. Su TTL es el tiempo máximo de uso desde el momento en que cada resolver la aprendió. Por ello, no existe una cuenta atrás global común. Stub Resolver, reenviadores, resolvers recursivos y aplicaciones pueden descartar el mismo RRset antiguo en momentos diferentes. Un TTL reducido antes de una migración solo ayuda para respuestas recargadas posteriormente; los datos ya almacenados en caché no pueden retirarse.

También se almacena en caché la inexistencia. **NXDOMAIN** significa que el nombre consultado no existe; **NODATA** es una respuesta NOERROR en la que el nombre existe, pero no existe ningún RRset del tipo consultado. En ambas respuestas, el servidor autoritativo coloca su RR SOA en la Authority Section. El tiempo de caché negativa es el mínimo entre SOA-TTL y SOA.MINIMUM ([RFC 2308, secciones 3 a 5](https://datatracker.ietf.org/doc/html/rfc2308#section-3)). Esto explica por qué un selector DKIM o nombre de host recién creado puede seguir apareciendo inicialmente como inexistente después de un intento fallido previo.

`SERVFAIL` debe distinguirse de ello: el resolver no pudo producir una respuesta utilizable. Entre las causas se encuentran timeouts hacia servidores autoritativos, una delegación rota, un error de validación DNSSEC o una limitación interna de recursos. `REFUSED` significa, en cambio, que el servidor consultado no ejecuta la operación por su política. Por ello, una herramienta de diagnóstico debe mostrar RCODE, bit AA, servidor respondedor y secciones; una salida que solo indique «sin dirección» difumina diferencias decisivas.

## Transporte: UDP, TCP y rutas de resolver cifradas

Para este intercambio, la ruta de red debe permitir UDP y TCP en el puerto 53. TCP no está limitado a las transferencias de zona: un resolver puede utilizarlo directamente y debe poder recurrir a él tras una respuesta UDP truncada. Por ello, el bloqueo de TCP suele detectarse solo con RRsets grandes, ricos en DNSSEC o con muchas respuestas. Las consultas A pequeñas siguen funcionando y dan una impresión falsa ([RFC 7766](https://datatracker.ietf.org/doc/html/rfc7766)).

Sin EDNS, la carga útil DNS sobre UDP está limitada a 512 bytes. EDNS permite al solicitante anunciar una carga útil recibible mayor. Sin embargo, un valor excesivo puede forzar la fragmentación IP; si una ruta pierde o bloquea fragmentos, aparece el patrón típico de que las respuestas pequeñas funcionan y las grandes agotan el tiempo de espera. La especificación EDNS recomienda considerar la capacidad real de recepción y la ruta y, en caso de problemas, recurrir a valores menores o TCP ([RFC 6891, sección 6.2](https://datatracker.ietf.org/doc/html/rfc6891#section-6.2)).

[DNS over TLS](https://datatracker.ietf.org/doc/html/rfc7858), [DNS over HTTPS](https://datatracker.ietf.org/doc/html/rfc8484) o [DNS over QUIC](https://datatracker.ietf.org/doc/html/rfc9250) cifran una ruta de transporte DNS. Estos procedimientos no cambian el contenido de las zonas ni la delegación, y no sustituyen DNSSEC: el cifrado de transporte protege la conexión a un resolver, DNSSEC autentica los datos a lo largo de la cadena de delegación. El DNS clásico en el puerto 53 no está cifrado; la terminología común de los tipos de transporte está definida en [RFC 9499, sección 6](https://datatracker.ietf.org/doc/html/rfc9499#section-6).

## DNSSEC y cadena de confianza

DNSSEC añade a la resolución de nombres origen e integridad verificables. Una zona firma RRsets con RRSIG y publica las claves públicas como DNSKEY. El padre conecta la zona hija mediante DS a la cadena de confianza superior; NSEC o NSEC3 también pueden demostrar inexistencia. Un resolver validador comienza en su Trust Anchor y verifica esta cadena hasta la respuesta. El contenido sigue siendo público: DNSSEC no cifra ninguna consulta ([RFC 4033](https://datatracker.ietf.org/doc/html/rfc4033), [RFC 4034](https://datatracker.ietf.org/doc/html/rfc4034)).

Para la operación, no solo son relevantes los archivos de claves, sino varios estados acoplados temporalmente:

- Los RRSIG tienen inicio y vencimiento; una hora del sistema incorrecta o un fallo en la refirma puede hacer que toda una zona sea **bogus**.
- Un DS en el padre debe coincidir con una DNSKEY utilizable de la zona hija. Un DS huérfano es peor para los resolvers validadores que una delegación deliberadamente sin firmar.
- Durante los rollovers, publicación, firma, cambio en el padre, TTL y tiempos de caché deben planificarse como una máquina de estados.
- Un resolver validador suele devolver SERVFAIL para datos bogus. Una prueba sin validación puede mostrar simultáneamente una respuesta aparentemente normal.

El flag AD por sí solo es tan fiable como el resolver y la ruta hasta él. Para una comprobación independiente, un administrador debe examinar la cadena con una herramienta validadora y localizar la transición defectuosa entre DS, DNSKEY y RRSIG.

## Operación autoritativa y flujo de datos

Detrás de la respuesta autoritativa hay una ruta de distribución propia. En un modelo clásico, una fuente primaria gestiona la zona, DNS NOTIFY informa a los secundarios de una nueva SOA Serial y AXFR o IXFR transfieren datos completos o incrementales, respectivamente. Por tanto, un listener sano no demuestra aún que el servidor haya cargado la nueva zona. Deben considerarse conjuntamente Serial, estado de transferencia y respuesta de cada nodo autoritativo ([RFC 1996](https://datatracker.ietf.org/doc/html/rfc1996), [RFC 5936](https://datatracker.ietf.org/doc/html/rfc5936), [RFC 1995](https://datatracker.ietf.org/doc/html/rfc1995)).

La zona puede generarse a partir de archivos de texto, una base de datos, una API, Active Directory o una canalización Git/CI. Esta elección de implementación no modifica el protocolo DNS en red, pero determina límites transaccionales, auditabilidad y recuperación. RFC 2136 define actualizaciones dinámicas atómicas con Prerequisites; una llamada a una API de proveedor es, en cambio, un protocolo de control independiente y debe documentar sus propias reglas de consistencia y errores ([RFC 2136](https://datatracker.ietf.org/doc/html/rfc2136)).

Una copia de seguridad DNS solo es útil si permite restaurar la zona realmente servida. Según la plataforma, esto incluye:

- zona o base de datos fuente, incluidos SOA Serial y diario dinámico;
- configuración de servidor, Views, ACL, reenviadores y asignación de catálogos;
- secretos TSIG, claves privadas DNSSEC y estados de rollovers automáticos;
- datos padre fuera de la propia zona, en particular delegación, Glue y DS;
- un procedimiento probado para volver a aprovisionar secundarios y verificar semánticamente los datos de zona.

Los secundarios son copias de disponibilidad, pero no automáticamente una copia de seguridad histórica. Un cambio erróneo o malicioso puede replicarse rápidamente a todos los servidores autoritativos mediante NOTIFY y transferencia de zona.

## Pila de implementación y tecnología

DNS no designa un único daemon. La pila tecnológica común consta de nombres y RRsets, formato binario de consulta/respuesta, UDP y TCP, lógica de caché y DNSSEC opcional. Los servidores autoritativos, resolvers recursivos, reenviadores y DNS gestionado implementan estos componentes de maneras diferentes. Por ello, para la operación cuenta primero el rol de un producto y después su lenguaje o empaquetado:

| Implementación | Rol principal | Enfoque técnico |
|---|---|---|
| [BIND 9](https://bind9.readthedocs.io/en/latest/) | autoritativo y/o recursivo | servidor de nombres universal, archivos de zona, actualizaciones dinámicas, DNSSEC y herramientas de diagnóstico |
| [Unbound](https://unbound.docs.nlnetlabs.nl/en/latest/manpages/unbound.html) | resolver recursivo con caché | canalización modular de resolver, validador y caché; sin rol autoritativo principal |
| [Knot DNS](https://www.knot-dns.cz/) | autoritativo | solo autoritativo, procesamiento paralelo, transferencia de zona, DDNS y DNSSEC |
| [Windows Server DNS](https://learn.microsoft.com/windows-server/networking/dns/dns-overview) | autoritativo y recursivo | zonas opcionales integradas en AD, actualizaciones dinámicas seguras, políticas, caché y reenvío |

La separación de roles es más importante que el nombre del producto. Para una plataforma autoritativa pública, importan el aprovisionamiento de zonas, secundarios, firma DNSSEC y resiliencia DDoS; para un resolver empresarial, importan caché, reenvío, espacios de nombres internos, política, privacidad y validación.

## DNS en la operación de correo e identidad

En mensajería, una respuesta DNS se convierte directamente en una ruta de entrega. Un MTA remitente consulta los RRsets MX del dominio destinatario, prefiere el valor más bajo y trata las preferencias iguales como equivalentes. Solo si no existe MX, el propio dominio se considera destino implícito. Cuando hay registros MX, un valor A/AAAA en el Apex del dominio no es un sustituto. Cada destino MX necesita sus propias direcciones y no puede ser un alias; un dominio sin aceptación de correo publica Null MX `0 .` ([RFC 5321, sección 5](https://datatracker.ietf.org/doc/html/rfc5321#section-5), [RFC 2181, sección 10.3](https://datatracker.ietf.org/doc/html/rfc2181#section-10.3), [RFC 7505](https://datatracker.ietf.org/doc/html/rfc7505)).

Los registros PTR se resuelven mediante el árbol de direcciones inverso. Las zonas directa e inversa suelen tener propietarios diferentes; los cambios deben coordinarse entre el operador del dominio y el de la dirección IP. Un PTR es un nombre, no una prueba criptográfica de identidad. Las plataformas de correo receptoras pueden usar una resolución directa/inversa coherente como señal, pero su política concreta de reputación o aceptación no es una propiedad de DNS.

SPF, DKIM y DMARC utilizan DNS como canal de publicación, pero definen sus propias reglas de evaluación. SPF lee una única carga TXT lógica y limita los términos que causan DNS; DKIM dirige las claves mediante selectores; DMARC se encuentra bajo `_dmarc`. Estos procedimientos se tratan técnicamente en [SPF, DKIM y DMARC](/kb/mail-auth). Para ellos, el administrador DNS debe dominar sobre todo correctamente Owner Name, división de cadenas TXT, tamaño de respuesta, TTL, delegación y tiempos de caché negativa.

También [LDAP](/kb/ldap) y [Kerberos](/kb/kerberos) utilizan frecuentemente registros SRV para el descubrimiento de servicios. Un registro SRV contiene, además de destino y puerto, una prioridad y un peso. Estos valores no son una configuración universal de balanceador de carga; solo los clientes que implementan el procedimiento SRV correspondiente los interpretan.

## Diagnóstico

Por ello, el diagnóstico nunca comienza con «DNS no funciona», sino con nombre, tipo, clase, servidor consultado, transporte y momento. Los resolvers empresariales, los resolvers públicos y los servidores autoritativos pueden proporcionar temporalmente o por Split DNS y política respuestas diferentes. Esta divergencia no es ruido de medición, sino la indicación más importante de en qué punto se separa la ruta de resolución.

### Consultar RRsets de forma específica

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für gezielte DNS-Abfragen">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux y Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$server = "192.0.2.53"
Resolve-DnsName example.ch. -Type SOA -Server $server -DnsOnly
Resolve-DnsName example.ch. -Type MX -Server $server -DnsOnly
Resolve-DnsName _dmarc.example.ch. -Type TXT -Server $server -DnsOnly
Resolve-DnsName 25.113.0.203.in-addr.arpa. -Type PTR -Server $server -DnsOnly</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">server=192.0.2.53
dig @"$server" example.ch. SOA +noall +answer +authority
dig @"$server" example.ch. MX +noall +answer +authority
dig @"$server" _dmarc.example.ch. TXT +noall +answer +authority
dig @"$server" -x 203.0.113.25 +noall +answer +authority</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) puede forzar un servidor específico, tipo de registro, solo DNS y TCP. [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) muestra además flags, secciones, RCODE, servidor respondedor y tiempo de consulta. Sin un `-Server` o `@server` explícito, se prueba el resolver configurado, no necesariamente la fuente autoritativa.

### Comprobar TCP y DNSSEC por separado

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DNS-Transport- und DNSSEC-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux y Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$server = "192.0.2.53"
Resolve-DnsName example.ch. -Type MX -Server $server -DnsOnly -TcpOnly
Resolve-DnsName example.ch. -Type DNSKEY -Server $server -DnsOnly -DnssecOk</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">server=192.0.2.53
dig @"$server" example.ch. MX +tcp
dig @"$server" example.ch. DNSKEY +dnssec
delv @"$server" example.ch. MX</code></pre>
  </div>
</div>

[`delv`](https://bind9.readthedocs.io/en/latest/manpages.html#delv-dns-lookup-and-validation-utility) utiliza la lógica de resolver y validador de BIND para comprobar una cadena DNSSEC. En cambio, una consulta correcta con `+dnssec` solo demuestra que se solicitaron y entregaron datos DNSSEC; no valida la cadena automáticamente. La prueba TCP separada revela firewalls que permiten UDP/53 pero bloquean TCP/53.

### Vaciar selectivamente cachés locales

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem zum Leeren des lokalen DNS-Caches">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux y Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Clear-DnsClientCache</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sudo resolvectl flush-caches
resolvectl statistics</code></pre>
  </div>
</div>

[`Clear-DnsClientCache`](https://learn.microsoft.com/powershell/module/dnsclient/clear-dnsclientcache) vacía la caché del cliente DNS de Windows. [`resolvectl`](https://man7.org/linux/man-pages/man1/resolvectl.1.html) controla la caché de `systemd-resolved`; en sistemas con `nscd`, `dnsmasq`, un Unbound local o una caché de aplicación, otra caché es responsable. Vaciar la caché del cliente nunca modifica la caché de un resolver upstream.

### Asignar el patrón de error al punto de transferencia

| Observación | Nivel probable | Siguiente comprobación |
|---|---|---|
| un resolver proporciona un valor antiguo y los servidores autoritativos el nuevo | caché positiva | TTL restante, cadena de reenviadores, caché de aplicación |
| NXDOMAIN persiste tras crear el registro | caché negativa o zona incorrecta | SOA en respuesta negativa, delegación padre, Owner Name |
| NOERROR sin Answer | el nombre existe, falta el tipo solicitado | cadena CNAME, QTYPE exacto, SOA NODATA |
| solo agotan el tiempo de espera respuestas grandes o firmadas | EDNS, fragmentación o fallback a TCP | `TC`, tamaño EDNS menor, prueba TCP explícita, firewall |
| el resolver validador devuelve SERVFAIL y uno no validador una respuesta | DNSSEC bogus | coincidencia DS/DNSKEY, tiempos RRSIG, algoritmo, hora del sistema |
| los servidores autoritativos entregan Serials diferentes | replicación | NOTIFY, IXFR/AXFR, ACL de transferencia, diario, fuente primaria |
| clientes públicos e internos ven destinos diferentes | Split DNS o política de resolver | resolver consultado, asignación de View, subred del cliente, reenviador |
| existe MX, pero la entrega falla antes de SMTP | nombre de destino derivado o transporte | MX Preference, A/AAAA del destino MX, TCP/25, prohibición de CNAME |

## Monitorización y criterios operativos

La monitorización debe representar la misma ruta. Una única búsqueda A contra el resolver local predeterminado no detecta una delegación rota, TCP bloqueado, firmas vencidas ni un secundario con Serial antiguo. Por ello, para mensajería e identidad deben recopilarse por separado al menos las siguientes señales:

- respuestas autoritativas de cada NS publicado mediante UDP y TCP, incluidos bit AA y SOA Serial;
- delegación padre, RRset NS hijo, Glue y, para zonas firmadas, la transición DS/DNSKEY;
- latencia recursiva, proporción de aciertos de caché y tasas de timeout, SERVFAIL, REFUSED y NXDOMAIN;
- tamaño de respuesta, truncamiento, errores EDNS y fallback a TCP;
- momentos de vencimiento de RRSIG, estado de rollover de claves y colas de firma;
- éxito de NOTIFY, AXFR e IXFR, así como antigüedad de la zona en cada secundario;
- RRsets funcionales como MX, direcciones A/AAAA asociadas, PTR y los nombres TXT necesarios para la [autenticación de correo](/kb/mail-auth).

Una prueba sintética debe comprobar tanto la ruta normal del cliente como la fuente autoritativa. Solo así puede distinguirse si una incidencia reside en los datos publicados, la delegación, una caché de resolver o la aplicación.

## Seguridad y dominios de fallo

Los dos roles de servidor requieren medidas de protección diferentes. La recursión debe pertenecer solo a clientes de confianza; un resolver abierto puede utilizarse para ataques de reflexión y amplificación. Los servidores autoritativos, en cambio, deben seguir siendo accesibles mundialmente, pero no deben resolver recursivamente nombres ajenos arbitrarios. Mezclar roles amplía la superficie de ataque y dificulta identificar las causas de carga ([RFC 5358](https://datatracker.ietf.org/doc/html/rfc5358)).

DNSSEC protege los RRsets publicados contra modificaciones inadvertidas, no el proceso del servidor ni la disponibilidad. **TSIG** autentica mensajes DNS individuales con una clave compartida y se utiliza, entre otras cosas, para actualizaciones y transferencias de zona; no es una firma pública de los datos de zona ([RFC 8945](https://datatracker.ietf.org/doc/html/rfc8945)). Las ACL de transferencia, los secretos TSIG y las claves privadas DNSSEC son objetos de protección separados.

Split DNS y las respuestas de política pueden ser necesarios, pero crean varias verdades para el mismo QNAME/QTYPE. Deben documentarse como mínimo el criterio de asignación, la zona fuente, la ruta de reenvío, el comportamiento DNSSEC y la monitorización por View. De lo contrario, una divergencia intencionada se interpretará en la siguiente incidencia como un error de caché o un problema de replicación.

## Historia técnica

Antes de DNS, Internet distribuía tablas de hosts mantenidas centralmente. Con más redes, hosts y operadores independientes, este procedimiento se convirtió en un cuello de botella. Paul Mockapetris describió en 1983 un servicio de nombres jerárquico y delegable en RFC 882 y RFC 883. RFC 1034 y RFC 1035 sustituyeron esta versión en 1987 y, como STD 13, siguen constituyendo el núcleo del sistema.

La primera implementación funcional de servidor, **Jeeves**, se ejecutó en 1983/84 en sistemas DEC-TOPS-20. Poco después surgió en la University of California, Berkeley, bajo financiación de DARPA, el Berkeley Internet Name Domain Package **BIND** para Unix. BIND 8 apareció en 1997 y BIND 9 en septiembre de 2000 como una reimplementación amplia. ISC documenta esta evolución en [A Brief History of the DNS and BIND](https://bind9.readthedocs.io/en/latest/history.html).

El protocolo creció gradualmente sin sustituir el núcleo jerárquico: NOTIFY e IXFR aceleraron la sincronización de zonas en la década de 1990, EDNS amplió el modelo de mensajes en 1999 y posteriormente se consolidó en RFC 6891, DNSSEC recibió en 2005 los mecanismos hoy fundamentales DNSKEY/DS/RRSIG, y se añadieron transportes de resolver cifrados con DoT, DoH y DoQ. Por ello, DNS no es un protocolo congelado de 1987, sino un sistema ampliable cuya compatibilidad hacia atrás y largos estados de caché caracterizan operativamente cada cambio.

## Fuentes

- [RFC Editor – STD 13: Domain Name System](https://www.rfc-editor.org/info/std13)
- [RFC 9499 – DNS Terminology](https://datatracker.ietf.org/doc/html/rfc9499) – terminología actual sobre roles, zonas, caché, DNSSEC y transporte.
- [RFC 1034 – Domain Names: Concepts and Facilities](https://datatracker.ietf.org/doc/html/rfc1034) – espacio de nombres, zonas, delegación, resolvers y roles de servidor.
- [RFC 1035 – Domain Names: Implementation and Specification](https://datatracker.ietf.org/doc/html/rfc1035) – formato de red, Resource Records, secciones de mensajes y archivos maestros.
- [RFC 6891 – Extension Mechanisms for DNS (EDNS(0))](https://datatracker.ietf.org/doc/html/rfc6891) – OPT, flags ampliados y carga útil UDP.
- [RFC 2181, sección 5](https://datatracker.ietf.org/doc/html/rfc2181)
- [`AAAA`](https://datatracker.ietf.org/doc/html/rfc3596)
- [RFC 2782 – A DNS RR for specifying the location of services](https://datatracker.ietf.org/doc/html/rfc2782) – estructura y selección de registros SRV.
- [RFC 8659 – DNS Certification Authority Authorization](https://datatracker.ietf.org/doc/html/rfc8659) – registros CAA y evaluación por autoridades de certificación.
- [RFC 2308 – Negative Caching of DNS Queries](https://datatracker.ietf.org/doc/html/rfc2308) – NXDOMAIN, NODATA, SOA y tiempo de caché negativa.
- [RFC 7766 – DNS Transport over TCP](https://datatracker.ietf.org/doc/html/rfc7766) – compatibilidad TCP obligatoria y comportamiento de conexión.
- [RFC 7858 – DNS over TLS](https://datatracker.ietf.org/doc/html/rfc7858) – DNS sobre TLS.
- [RFC 8484 – DNS Queries over HTTPS](https://datatracker.ietf.org/doc/html/rfc8484) – DNS sobre HTTPS.
- [RFC 9250 – DNS over Dedicated QUIC Connections](https://datatracker.ietf.org/doc/html/rfc9250) – DNS sobre QUIC.
- [RFC 4033 – DNS Security Introduction and Requirements](https://datatracker.ietf.org/doc/html/rfc4033) – objetivos de protección DNSSEC, validación y límites.
- [RFC 4034 – Resource Records for DNSSEC](https://datatracker.ietf.org/doc/html/rfc4034) – DNSKEY, DS, RRSIG y NSEC.
- [RFC 1996 – DNS NOTIFY](https://datatracker.ietf.org/doc/html/rfc1996) – notificación a servidores secundarios.
- [RFC 5936 – DNS Zone Transfer Protocol (AXFR)](https://datatracker.ietf.org/doc/html/rfc5936) – transferencias completas de zona mediante TCP.
- [RFC 1995 – Incremental Zone Transfer (IXFR)](https://datatracker.ietf.org/doc/html/rfc1995) – sincronización incremental de zonas.
- [RFC 2136 – Dynamic Updates in DNS](https://datatracker.ietf.org/doc/html/rfc2136) – cambios atómicos con Prerequisites.
- [BIND 9](https://bind9.readthedocs.io/en/latest/)
- [Unbound Documentation – unbound(8)](https://unbound.docs.nlnetlabs.nl/en/latest/manpages/unbound.html) – caché recursiva y validación DNSSEC.
- [Knot DNS](https://www.knot-dns.cz/) – implementación solo autoritativa y funciones operativas.
- [Microsoft Learn – DNS in Windows Server](https://learn.microsoft.com/windows-server/networking/dns/dns-overview) – roles DNS de Windows, integración AD, caché y reenvío.
- [RFC 5321, sección 5](https://datatracker.ietf.org/doc/html/rfc5321)
- [RFC 7505 – Null MX](https://datatracker.ietf.org/doc/html/rfc7505) – identificación explícita de dominios sin recepción de correo.
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) – consultas DNS en Windows.
- [ISC BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html) – opciones de consulta, selección de servidor e interpretación de salida.
- [`Clear-DnsClientCache`](https://learn.microsoft.com/powershell/module/dnsclient/clear-dnsclientcache)
- [`resolvectl`](https://man7.org/linux/man-pages/man1/resolvectl.1.html)
- [RFC 5358 – Preventing Use of Recursive Nameservers in Reflector Attacks](https://datatracker.ietf.org/doc/html/rfc5358) – limitación de recursión y protección contra abuso.
- [RFC 8945 – Secret Key Transaction Authentication for DNS (TSIG)](https://datatracker.ietf.org/doc/html/rfc8945) – autenticación de mensajes para actualizaciones y transferencias.
- [ISC – A Brief History of the DNS and BIND](https://bind9.readthedocs.io/en/latest/history.html) – Jeeves, Berkeley, BIND 8 y BIND 9.
