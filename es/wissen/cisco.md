---
title: "Cisco Secure Email: Gateway, AsyncOS y SMA"
blatt: "cisco"
description: "Cisco Secure Email para administradores de mensajería: canalización de correo SEG/ESA, listeners, HAT y RAT, cola de trabajo y entrega, políticas de correo, AsyncOS, servicios SMA, límites de clúster, pila tecnológica, monitorización, recuperación y diagnóstico."
fakten:
  - label: Roles de producto
    wert: Secure Email Gateway (SEG/ESA) · Secure Email and Web Manager (SMA)
    href: https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html
  - label: Rol del sistema
    wert: Gateway de correo SMTP delante o entre sistemas de correo
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Sistema operativo
    wert: Cisco AsyncOS
    href: https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html
  - label: Canalización
    wert: Receipt → Work Queue → Delivery
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Aceptación
    wert: Listener · HAT · Sender Groups · RAT · LDAP
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Procesamiento
    wert: Message Filters · Mail Policies · Content Filters · Scan-Engines
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html
  - label: Servicios centrales
    wert: Tracking · Reporting · Cuarentenas de spam y políticas en SMA
    href: https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html
  - label: Clúster de configuración
    wert: peer-to-peer; sin HA de colas ni tráfico
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html
  - label: Factores de forma
    wert: appliance virtual · nube pública · Cisco Cloud Gateway
    href: https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html
  - label: Administración
    wert: interfaz web · CLI mediante SSH · API REST
    href: https://docs.ces.cisco.com/docs/api
  - label: Estados principales
    wert: configuración · cola · cuarentenas · tracking/reporting · claves y certificados
    href: https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0100010.html
  - label: Origen
    wert: tecnología IronPort; adquirida por Cisco en 2007
    href: https://newsroom.cisco.com/c/r/newsroom/en/us/a/y2007/m01/cisco-announces-agreement-to-acquire-ironport.html
werbung:
  - tools
  - newsletter
ctaThemen:
  - cisco-esa-sma
translationSourceHash: c796a2c50226bbdcf5a0d7a7562913a5126b28cdbd05a5b72065afcf531bfd42
translationModel: gpt-5.6-terra
translatedAt: 2026-10-03T09:43:32.701Z
translationReview: automatic
---

# Cisco Secure Email: Gateway, AsyncOS y SMA

**Cisco Secure Email Gateway**, abreviado SEG e históricamente **Email Security Appliance** o ESA, es una puerta de enlace de correo [SMTP con estado](/kb/smtp). Termina las sesiones SMTP entrantes, decide su aceptación, procesa los mensajes en una Work Queue interna y abre una nueva sesión SMTP para la entrega. Por tanto, el límite de responsabilidad técnica no está en el handshake TCP o TLS correcto, sino en la respuesta SMTP positiva tras el contenido del mensaje: desde ese momento, la puerta de enlace debe entregar o generar un error conforme a la norma ([Cisco: Email Pipeline](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

La segunda función clásica es **Cisco Secure Email and Web Manager**, SMA. Normalmente no actúa como MTA regular en la ruta de correo de producción. Gestiona datos centralizados de tracking y reporting y, según el diseño, cuarentenas de spam, políticas, virus y brotes de varios gateways. Por ello, una caída del SMA puede dejar intacta la entrega en los nodos SEG y, al mismo tiempo, interrumpir las búsquedas, la cuarentena de usuarios finales o el procesamiento de mensajes retenidos ([Cisco: SMA Message Tracking](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html), [Cisco: Centralized Quarantines](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0110.html)).

Ambas funciones se ejecutan en **AsyncOS**, una plataforma de software mantenida por Cisco como unidad de appliance. Los administradores no gestionan los paquetes subyacentes como en un servidor Linux general; la superficie técnica fiable consta de configuración de AsyncOS, CLI, interfaz web, API REST, suscripciones de logs, MIB, canales de actualización e integraciones documentadas. Los resúmenes de código abierto de Cisco prueban numerosos componentes integrados, pero no una lista de materiales públicamente mantenible de la canalización de correo propietaria. Por tanto, las bibliotecas individuales no deben equipararse con la arquitectura completa ([Cisco: Open Source Used in AsyncOS](https://www.cisco.com/c/dam/en_us/about/doing_business/open_source/docs/CiscoEmailSecurityAppliance-AsyncOS1253-1688323737.pdf), [Cisco: AsyncOS API](https://docs.ces.cisco.com/docs/api)).

La explicación acompaña un mensaje a través de Cisco Secure Email: desde el listener, HAT, RAT y Work Queue hasta la entrega. Después se tratan SMA, operación en clúster, dependencias, diagnóstico y recuperación.

## Roles de producto y límites de confianza

Un diseño local típico coloca al menos dos nodos SEG en la DMZ y un SMA en una red de administración interna. Los MX de DNS o un servicio previo distribuyen las conexiones entrantes entre los gateways; para el tráfico saliente, los conectores de smarthost del sistema de correo determinan la ruta por el gateway. Varios nodos SEG solo son de alta disponibilidad cuando DNS, el balanceador de carga o el MTA emisor pueden utilizar destinos alternativos. El clúster de configuración de AsyncOS por sí solo no asume este encaminamiento de tráfico ([Cisco: Centralized Management Using Clusters](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html), [RFC 5321, Address Resolution](https://datatracker.ietf.org/doc/html/rfc5321)).

Cisco documenta appliances virtuales, despliegues en nube pública y un Secure Email Cloud Gateway operado. Estas variantes comparten términos de producto, pero desplazan responsabilidades: con la appliance virtual, el cliente es responsable del hipervisor, la red, la capacidad y la restauración; con Cloud Gateway, Cisco proporciona la infraestructura de gateway. **Secure Email Threat Defense** es, en cambio, una plataforma nativa de nube para análisis y protección que puede integrarse mediante gateway, journaling o API de Microsoft. No es sinónimo de la Work Queue local ni un término alternativo para SMA ([Cisco: Secure Email Data Sheet](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html), [Cisco: Email Threat Defense Data Sheet](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/secure-email-threat-defense-ds.html)).

| Rol | En la ruta SMTP | Estado persistente | Efecto de una caída |
|---|---:|---|---|
| Secure Email Gateway | sí | Cola, cuarentenas locales, configuración, certificados, logs | Aceptación o entrega interrumpida en este nodo |
| Secure Email and Web Manager | normalmente no | Tracking, reporting, cuarentenas centralizadas, listas seguras/de bloqueo, configuración propia | Visibilidad y servicios centrales de cuarentena afectados |
| Email Threat Defense | depende de la integración | Telemetría, investigación y políticas en la nube | Análisis o remediación adicionales afectados |
| Sistema de correo | antes o después del gateway | Buzones, colas de transporte, conectores | Acceso de usuarios o entrega de extremo a extremo afectados |

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-cisco.svg?v=20260813" title="Interaktive Infografik: Cisco Secure Email mit SEG-Mailpipeline, Listener, HAT und RAT, Work Queue, Delivery, SMA-Diensten, Konfigurationscluster und Admin-Kontrollpunkten" loading="lazy">
  <a href="/images/kb-interaktiv-cisco.svg?v=20260813">Abrir directamente el gráfico interactivo</a>.
</iframe>

## Receipt: listener, HAT y RAT

Un **listener** vincula SMTP a una interfaz IP y constituye el primer límite de políticas. Los listeners públicos suelen aceptar tráfico de Internet para dominios locales; los listeners privados reciben mensajes salientes desde redes controladas. Estas funciones son configuración, no una propiedad inherente de confianza del puerto. Un listener privado con una autorización de relay demasiado amplia es un relay abierto, aunque tenga un nombre interno.

La **Host Access Table**, HAT, asigna hosts conectados a Sender Groups. Sus Mail Flow Policies determinan, entre otros aspectos, si una conexión se acepta, rechaza, limita o procesa sin determinados escaneos. La **Recipient Access Table**, RAT, define dominios de destinatarios locales para mensajes entrantes. Opcionalmente, [LDAP](/kb/ldap) comprueba destinatarios concretos durante la sesión SMTP o posteriormente en la Work Queue; como alternativa, SMTP Call-Ahead puede consultar al servidor posterior. Cisco separa así cuatro identidades que no deben confundirse durante una incidencia: IP de origen, remitente del sobre, destinatario del sobre e identidades de cabecera ([Cisco: Email Pipeline, Incoming](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html)).

Una documentación limpia del listener incluye, como mínimo por dirección, IP y puerto de enlace, redes de origen permitidas, nombres EHLO esperados, modo TLS, requisito de certificado de cliente, orden de HAT, dominios RAT, validación de destinatarios, tamaño máximo de mensaje, límites de tasa y perfil de rebote. El orden es especialmente crítico: un rechazo HAT temprano no genera un registro de Message ID como un mensaje aceptado posteriormente; por ello, un equipo de soporte no puede encontrarlo con la misma búsqueda.

Tras la aceptación SMTP comienza el procesamiento real de contenido y políticas. Su orden es importante porque un resultado temprano puede influir en comprobaciones posteriores, grupos de destinatarios o rutas de entrega.

## Work Queue: orden, splintering y políticas

Tras la aceptación, el mensaje llega a la **Work Queue**. Cisco documenta allí routing y masquerading, Message Filters, listas seguras/de bloqueo, antispam, antivirus, Graymail, reputación y análisis de archivos, Content Filters, Outbreak Filters y cuarentenas. El orden forma parte del modelo de seguridad. Por lo general, un cambio de política no afecta retroactivamente a mensajes ya almacenados; por tanto, activar posteriormente un escáner no corrige automáticamente una omisión previa ([Cisco: Email Pipeline, Work Queue](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html)).

Los **Message Filters** operan antes de la Mail Policy relacionada con el destinatario y pueden modificar, archivar, poner en cuarentena, rebotar o descartar mensajes en función del sobre, cabeceras, contenido, adjuntos o datos de conexión. Posteriormente, AsyncOS puede **fragmentar** un mensaje con varios destinatarios: se generan Message IDs separados para políticas de destinatarios diferentes y, por tanto, estados finales distintos. En consecuencia, un único valor original de inyección o ICID puede ramificarse en varios MID y resultados de entrega. El tracking debe mostrar este árbol, no limitarse a buscar por asunto.

Las **Mail Policies** controlan filtros de escaneo y contenido basados en destinatarios o remitentes. Según Cisco, DLP se limita al procesamiento saliente. Las licencias, el estado de actualización de los motores y la conectividad en la nube también determinan qué comprobaciones se realizan realmente. Para cada política, una prueba sólida requiere un caso positivo inocuo, un caso negativo específico y el estado final esperado: entrega, modificación, cuarentena, descarte o rebote.

## Delivery: SMTP Routes, Destination Controls y cola

En la fase Delivery, AsyncOS selecciona la ruta, la interfaz de origen y el destino. Las **SMTP Routes** reemplazan la resolución MX normal para dominios configurados; los **Destination Controls** limitan conexiones paralelas y destinatarios por destino. Los Virtual Gateways pueden proporcionar distintas direcciones IP de origen, nombres de host y colas de entrega. Estos ajustes influyen en la reputación, SPF, inclusión en listas permitidas por pares y el lugar donde espera un mensaje ([Cisco: Email Pipeline, Delivery](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html), [RFC 7208](https://datatracker.ietf.org/doc/html/rfc7208)).

TLS saliente es hop-by-hop. AsyncOS puede utilizar [STARTTLS](/kb/tls) con los pares; que funcione protege este tramo de transporte, pero no dice nada sobre saltos anteriores o posteriores. Para políticas forzadas deben documentarse patrones de destino, validación de certificados, relación con el nombre y comportamiento ante errores. TLS oportunista puede volver a texto plano si falla el handshake; una política obligatoria debe, en cambio, poner en cola o fallar ([Cisco: Verify and Troubleshoot TLS Certificates](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118844-technote-esa-00.html), [RFC 3207](https://datatracker.ietf.org/doc/html/rfc3207)).

La antigüedad de la cola es más importante que su tamaño puro. Un volumen elevado puede ser saludable con un alto rendimiento; pocos mensajes muy antiguos indican un bloqueo persistente de destino, DNS, TLS o políticas. Para cada ruta, la imagen del incidente debe incluir el mensaje más antiguo, el motivo del reintento, el siguiente intento, la respuesta del destino y el par responsable.

En cuanto la ESA ha reenviado un mensaje o lo ha puesto en cuarentena, parte de la perspectiva administrativa se traslada al SMA. Sin embargo, no sustituye los datos locales de cola y sistema de la ESA.

## SMA: tracking, reporting y cuarentenas

El SMA recopila datos de tracking y reporting de varios nodos SEG. Message Tracking puede mostrar estados finales como `Delivered`, `Dropped`, `Bounced`, `Quarantined`, `Queued`, `Processing` y `Splintered`. Sin embargo, es un índice derivado: si faltan datos de exportación, un servicio se retrasa o el mensaje queda fuera de retención, un resultado vacío no demuestra que el mensaje nunca se procesara. La evidencia principal sigue siendo los logs de correo correspondientes y la cadena MID en el SEG ([Cisco: Tracking Messages](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html)).

La cuarentena de spam y las cuarentenas de políticas, virus y brotes son servicios separados con distintos usuarios, vías de liberación y retenciones. Las cuarentenas centralizadas almacenan mensajes en el SMA detrás del firewall y pueden incluirse en su copia de seguridad estándar. Con una ocupación del 75, 85 y 95 por ciento, AsyncOS genera alertas de umbral documentadas. Si un servicio central de cuarentena deja de estar disponible, la operación necesita una decisión previamente probada: poner temporalmente en cola, cambiar al procesamiento local o desactivar de forma controlada la política correspondiente ([Cisco: Centralized Quarantines](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0110.html), [Cisco: Centralizing Services](https://www.cisco.com/c/en/us/td/docs/security/esa/esa13-7-0/user_guide/b_ESA_Admin_Guide_13-7/b_ESA_Admin_Guide_12_1_chapter_0101011.html)).

## Un clúster de configuración no es HA de correo

AsyncOS puede conectar varios gateways en un **clúster de configuración** peer-to-peer. Los ajustes se pueden mantener a nivel de clúster, grupo o máquina; no existe un nodo de clúster primario. Los miembros deben utilizar una versión compatible de AsyncOS y comunicarse mediante SSH o Cluster Communication Service. El clúster replica configuración, no sesiones SMTP activas, contenidos de cola, cuarentenas locales ni progreso de entrega ([Cisco: Centralized Management Using Clusters](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html)).

Por ello, existen tres mecanismos separados:

- **Distribución de tráfico:** varios destinos MX, balanceadores de carga o failover de smarthost;
- **Consistencia de configuración:** clúster AsyncOS con overrides claros de clúster, grupo y máquina;
- **Disponibilidad de datos:** estado de cola por SEG y datos de tracking y cuarentena en el SMA.

La pérdida de un nodo tras una aceptación SMTP positiva puede afectar a mensajes que solo están en su cola local. El remitente no debe reenviarlos sin más mientras el estado original de entrega no esté claro; de lo contrario se generan duplicados. Por tanto, una prueba de recuperación no solo debe cargar la configuración, sino rastrear mensajes de prueba aceptados a través de una caída controlada de nodo.

## Pila tecnológica y superficies de administración

AsyncOS es una plataforma de appliance cerrada. Cisco publica avisos de código abierto para los componentes incluidos, pero no un plan completo de código fuente o lenguajes de los servicios propietarios. Afirmaciones como «escrito en Python» o «basado en FreeBSD» no constituyen información operativa fiable sin evidencia del fabricante específica de la versión. Para los administradores es más relevante la siguiente pila verificable ([Cisco: Open Source Used in AsyncOS](https://www.cisco.com/c/dam/en_us/about/doing_business/open_source/docs/CiscoEmailSecurityAppliance-AsyncOS1253-1688323737.pdf)):

| Nivel | Tecnología verificable | Relevancia operativa |
|---|---|---|
| Transporte de correo | Listener SMTP, Receipt, Work Queue, Delivery Queue | Límite de aceptación, orden de políticas, reintentos y rebotes |
| Políticas y análisis | HAT/RAT, Message Filters, Mail Policies, Content Filters, Scan-Engines | Orden, licencias, actualizaciones de motores, splintering |
| Datos y búsqueda | Colas/cuarentenas locales; tracking, reporting y cuarentenas centralizadas de SMA | Capacidad, retención, copia de seguridad y protección de datos |
| Administración | GUI HTTPS, CLI interactiva mediante SSH, configuración XML | Cambios, commit, exportación, restauración y auditoría |
| Automatización | API RESTful de AsyncOS con Swagger | Reporting, tracking y acceso a cuarentenas; sin configuración completa no verificada |
| Telemetría | Mail Logs, otras Log Subscriptions, Syslog, Alerts, SNMP/MIB, API | Correlación mediante ICID/MID/DCID y estado de recursos |
| Plataforma | Appliance de hardware, virtual y en la nube | Responsabilidad de cómputo, almacenamiento, red y ciclo de vida |

Los cambios en la CLI siguen un patrón transaccional: los comandos modifican primero una configuración en ejecución, `commit` la activa y `clearchanges` la descarta. Un runbook debe indicar el diálogo completo y el modo de configuración; los fragmentos de copiar y pegar son peligrosos debido a diferencias entre versiones y clústeres. La API REST proporciona acceso autenticado de forma segura a informes, contadores, datos de tracking y cuarentena; su interfaz Swagger local documenta el alcance de API realmente instalado ([Cisco: AsyncOS API](https://docs.ces.cisco.com/docs/api), [Cisco: SEG Support Documentation](https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html)).

## Dependencias de red, identidad y tiempo

Una vez establecida la canalización de correo, se pueden revisar sus conexiones externas. Cada fila muestra qué sistema inicia una conexión, para qué se necesita y cómo se manifiesta un error.

| Conexión | Puerto habitual | Iniciador | Finalidad y síntoma de error |
|---|---:|---|---|
| SMTP | TCP 25 | MTA externo, sistema de correo interno o SEG | Aceptación y reenvío; timeout, 4xx/5xx, crecimiento de cola |
| HTTPS | TCP 443 o configurado | Administrador, usuario final o cliente API | GUI, API, cuarentena; comprobar por separado certificado, SSO y roles |
| SSH | TCP 22 o configurado | Administrador o miembro SEG | CLI y, opcionalmente, comunicación de clúster |
| CCS | TCP 2222 de forma predeterminada, configurable | Miembro SEG | Clúster de configuración; sin flujo de correo |
| DNS | UDP/TCP 53 | SEG/SMA | MX, A/AAAA, PTR, reputación y actualizaciones |
| LDAP/LDAPS | TCP 389/636 | SEG/SMA | Destinatarios, routing, grupos y autenticación de administradores |
| Syslog | UDP/TCP 514 o TLS 6514 según diseño | SEG/SMA | Transporte externo de logs; definir modelo de pérdida y backpressure |
| SNMP | UDP 161/162 | Monitorización o appliance | Consulta de estado y traps; preferir SNMPv3 |

Los números de puerto por sí solos no prueban una función activa. Las asignaciones proceden de la [IANA Service Name and Port Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml); Cisco documenta CCS y su configurabilidad en el capítulo de clústeres. Los firewalls deben incluir origen, destino, dirección, protocolo, requisitos de TLS o autenticación y finalidad empresarial.

[LDAP](/kb/ldap) puede alimentar la aceptación de destinatarios, routing, pertenencia a grupos y autenticación de administradores. Estas consultas tienen esquemas, tiempos de espera y consecuencias de error diferentes. Si falla Recipient Acceptance, el sistema puede, según la configuración, rebotar con retraso o descartar; por ello, un error de autenticación en la GUI no demuestra un defecto en la comprobación SMTP de destinatarios. Las cuentas de servicio, Base DN, filtros, comportamiento de referrals, cadena de certificados y orden de failover deben documentarse por consulta ([Cisco: Email Pipeline, LDAP Recipient Acceptance](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html), [RFC 4511](https://datatracker.ietf.org/doc/html/rfc4511)).

DNS y la hora correcta son dependencias del sistema. La resolución de MX y hosts controla la entrega y accesibilidad del clúster; PTR y reputación influyen en la clasificación. NTP mantiene correlacionables las horas de logs, Received y tracking. Cisco exige nombres de host resolubles o direcciones IP utilizadas de forma consistente para clústeres, y describe la hora del sistema y NTP como parte de la configuración básica ([Cisco: Setup and Installation](https://www.cisco.com/c/en/us/td/docs/security/esa/esa13-7-0/user_guide/b_ESA_Admin_Guide_13-7/b_ESA_Admin_Guide_12_1_chapter_010.html)).

En caso de incidencias se sigue la misma ruta a la inversa: estado de entrega, decisión de Work Queue, política de Receipt, listener y dependencias de red.

## Monitorización y triaje de incidentes

La pregunta central es: **¿Ha aceptado el gateway el mensaje, lo ha procesado y a qué salto lo ha entregado?** Para ello, se encadenan identificadores de conexión, mensaje y entrega de los Mail Logs. Message Tracking en el SMA acelera la búsqueda, pero no sustituye los logs sin procesar. Las señales técnicas útiles son:

- tasa de aceptación, respuestas 4xx/5xx y conexiones rechazadas por listener y Sender Group;
- Work Queue y Delivery Queue, antigüedad del mensaje más antiguo y respuestas de destino recurrentes;
- duración de procesamiento y splintering, errores de Scan-Engine y antigüedad de actualizaciones;
- ocupación de cuarentenas locales y centralizadas, eventos de liberación y eliminación;
- valor de Resource Conservation, CPU, memoria, ocupación de disco y alertas críticas;
- disponibilidad y latencia de DNS, LDAP, SMA, servicios de actualización y nube;
- consistencia del clúster y Machine-Overrides no intencionados;
- vencimiento y uso de cada certificado TLS, así como cambios en el truststore.

En **Resource Conservation Mode**, AsyncOS limita progresivamente la aceptación para que la entrega pueda reducir el retraso; ante una escasez extrema de recursos no se aceptan mensajes nuevos. Por ello, el síntoma suele ser una disminución del rendimiento entrante, mientras que el desencadenante real es una ruta de destino lenta o un recurso lleno. Cisco proporciona estado y alertas en GUI/CLI; SNMPv3 y `ASYNCOS-MAIL-MIB` permiten monitorización externa ([Cisco: Resource Conservation](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117834-qanda-esa-00.html), [Cisco: SNMP Monitoring](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117831-qanda-esa-00.html)).

## Copia de seguridad, recuperación y actualización

Un archivo XML de configuración exportado es necesario, pero no es una copia de seguridad completa del sistema. Cisco documenta `saveconfig`, `mailconfig` y `loadconfig`; las frases de contraseña enmascaradas no se pueden volver a cargar. Los certificados y claves, el estado del clúster, Feature Keys, colas locales, cuarentenas locales, datos SMA y dependencias externas requieren evidencias propias ([Cisco: System Administration](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0100010.html), [Cisco: Automated Configuration Backup](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118403-technote-esa-00.html)).

| Objeto de recuperación | Copia de seguridad o reconstrucción | Prueba de aceptación |
|---|---|---|
| Configuración SEG | Exportación sin enmascarar, almacenada de forma protegida, más frases de contraseña documentadas | Cargar en instancia de reemplazo, diff y prueba de listener/política |
| Certificados y claves privadas | Copia de seguridad cifrada de claves, cadena CA y matriz de roles | Handshake HTTPS y SMTP-TLS con validación de nombre |
| Cola local | Normalmente no reconstruible desde la copia de seguridad de configuración | Caída de nodo con correo de prueba aceptado y control de duplicados |
| Datos SMA | Copia de seguridad SMA para tracking, reporting, cuarentenas y listas | Comprobar búsqueda, liberación de un correo de prueba y retención |
| Clúster | Exportación por nivel más Machine-Overrides documentados | Volver a conectar miembro y comprobar consistencia |
| Servicios externos | Configuración DNS, LDAP, Syslog, NTP, actualización y nube | Prueba sintética de extremo a extremo |

Las actualizaciones son migraciones de appliance. Antes deben comprobarse ruta de destino, estados intermedios compatibles, requisitos de hipervisor o nube, cambios de funciones, orden del clúster, espacio libre, tiempo de inactividad y límite de rollback. Las categorías de versión GD y MD de Cisco no son una recomendación automática para todos los entornos; son determinantes los Security Advisories, la matriz de soporte y el alcance de políticas propio probado. La página de soporte y las explicaciones de ciclo de vida deben formar parte del procedimiento de parcheo, no figurar como número de versión estático en el artículo ([Cisco: SEG Release Notes](https://www.cisco.com/c/en/us/support/security/email-security-appliance/products-release-notes-list.html), [Cisco: Software Lifecycle Support Statement](https://www.cisco.com/c/dam/en/us/td/docs/security/esa/lifecycle_support_statement/Secure_Email_Gateway_Software_Lifecycle_Support_Statement.pdf)).

## Herramientas de diagnóstico

La resolución de problemas sigue la ruta del mensaje desde fuera hacia dentro. Primero se comprueban nombres y accesibilidad, después aceptación SMTP, eventos de canalización, cola y, si procede, la evaluación SMA.

### DNS, MX y resolución de destino

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-Secure-Email-DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.ch
Resolve-DnsName seg1.example.ch -Type A,AAAA
Resolve-DnsName 192.0.2.25 -Type PTR
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short MX example.ch
dig +short A seg1.example.ch
dig +short AAAA seg1.example.ch
dig +short -x 192.0.2.25
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) y [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) muestran resolución MX, directa e inversa. La consulta debe repetirse desde la perspectiva de resolvedores internos y externos; las SMTP Routes de AsyncOS pueden anular el resultado MX visible.

### TCP y SMTP-TLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-Secure-Email-TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection seg1.example.ch -Port 25 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://seg1.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz seg1.example.ch 25
openssl s_client -starttls smtp -connect seg1.example.ch:25 \
  -servername seg1.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) solo prueban la ruta TCP. [`curl`](https://curl.se/docs/manpage.html) y [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) solicitan STARTTLS y muestran el handshake y la cadena de certificados; solo la validación esperada de nombre y confianza prueba la política TLS configurada.

### Mensaje de prueba SMTP autorizado

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-Secure-Email-SMTP-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
curl.exe --verbose --ssl-reqd --url smtp://seg1.example.ch:25 `
  --mail-from test-sender@example.ch `
  --mail-rcpt test-recipient@example.net `
  --upload-file .\seg-test.eml
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
swaks --server seg1.example.ch --port 25 --tls \
  --from test-sender@example.ch --to test-recipient@example.net \
  --data seg-test.eml
```

  </div>
</div>

[`curl`](https://curl.se/docs/manpage.html) y [`swaks`](https://jetmore.org/john/code/swaks/) envían un mensaje de prueba controlado. El remitente, destinatario y destino deben estar autorizados. Deben registrarse la respuesta SMTP final, ICID/MID, MID de fragmentación, política, estado de cuarentena o entrega y la llegada real.

### Interfaz web y API REST

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-AsyncOS-API-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Invoke-WebRequest -Method Head -Uri https://sma.example.ch/
Invoke-WebRequest -Method Head -Uri https://seg1.example.ch/swagger
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
curl --head --verbose https://sma.example.ch/
curl --head --verbose https://seg1.example.ch/swagger
```

  </div>
</div>

[`Invoke-WebRequest`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest) y [`curl`](https://curl.se/docs/manpage.html) comprueban accesibilidad HTTP y TLS. Un código de estado no prueba inicio de sesión, autorización de roles, importación de tracking ni función de cuarentena. La página Swagger solo describe la API de la instancia consultada; las pruebas de API de producción utilizan una cuenta mínima de solo lectura y no guardan tokens en el historial de shell.

### Ruta de paquetes en un punto de medición autorizado

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cisco-Secure-Email-Paketerfassung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
pktmon filter remove
pktmon filter add SEG-SMTP -p 25
pktmon start --capture --pkt-size 0 --file-name seg.etl
pktmon stop
pktmon pcapng seg.etl -o seg.pcapng
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
tcpdump -ni any -s 0 -w seg.pcap 'tcp port 25 or tcp port 443'
```

  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) y [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) solo ven el tráfico en el punto de medición elegido. Un cliente administrador no observa automáticamente la ruta entre el balanceador de carga, SEG, SMA y el MTA de destino. Los datos de paquetes pueden contener contenido SMTP antes de STARTTLS y metadatos personales, por lo que deben protegerse en consecuencia.

## Historia técnica

IronPort Systems desarrolló gateways de mensajería especializados y la línea de productos AsyncOS. Cisco anunció la adquisición de la empresa en enero de 2007 e integró su tecnología de seguridad de correo electrónico y web en su propia cartera de seguridad ([Cisco: Agreement to Acquire IronPort](https://newsroom.cisco.com/c/r/newsroom/en/us/a/y2007/m01/cisco-announces-agreement-to-acquire-ironport.html)). Posteriormente, los nombres de producto cambiaron de Cisco IronPort Email Security Appliance, pasando por Cisco Email Security Appliance, a **Cisco Secure Email Gateway**; términos históricos como ESA, C-Series y M-Series siguen siendo visibles en runbooks, mensajes de log, licencias y rutas de documentación.

La idea arquitectónica siguió siendo reconocible tras estos cambios de nombre: un gateway especializado con la canalización de tres etapas Receipt, Work Queue y Delivery, y un sistema de gestión separado para datos agregados y cuarentenas. Más tarde se añadieron appliances virtuales y de nube pública, Cloud Gateway, API REST y servicios de análisis basados en la nube. Email Threat Defense amplía la cartera con modelos de API, journaling y gateway; no modifica retroactivamente los límites de estado de una instalación ESA/SMA existente ([Cisco: Secure Email Data Sheet](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html), [Cisco: Email Threat Defense](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/secure-email-threat-defense-ds.html)).

Por ello, el nombre de una appliance instalada no basta como información de ciclo de vida. El modelo de hardware, plataforma virtual, rama AsyncOS, licencias activadas, actualizaciones de motores y reglas, así como servicios de nube dependientes, tienen ciclos de vida propios. Las páginas de soporte, versiones y fin de vida de Cisco son fuentes operativas dinámicas; un artículo estático debe enlazarlas, pero no fijar un estado de versión supuestamente siempre actual ([Cisco: SEG End-of-Life Notices](https://www.cisco.com/c/en/us/products/security/email-security-appliance/eos-eol-notice-listing.html), [Cisco: SEG Support](https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html)).

## Fuentes

- [Cisco – AsyncOS User Guide: Understanding the Email Pipeline](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_011.html)
- [IETF RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [Cisco – SMA User Guide: Tracking Messages](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0100.html)
- [Cisco – SMA User Guide: Centralized Quarantines](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_0110.html)
- [Cisco – Open Source Used in Email Security Appliance AsyncOS](https://www.cisco.com/c/dam/en_us/about/doing_business/open_source/docs/CiscoEmailSecurityAppliance-AsyncOS1253-1688323737.pdf)
- [Cisco – AsyncOS API](https://docs.ces.cisco.com/docs/api)
- [Cisco – AsyncOS User Guide: Centralized Management Using Clusters](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0101000.html)
- [Cisco – Secure Email Gateway and Secure Email and Web Manager Data Sheet](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/datasheet-c78-742868.html)
- [Cisco – Secure Email Threat Defense Data Sheet](https://www.cisco.com/c/en/us/products/collateral/security/cloud-email-security/secure-email-threat-defense-ds.html)
- [IETF RFC 7208 – Sender Policy Framework](https://datatracker.ietf.org/doc/html/rfc7208)
- [Cisco – Verify and Troubleshoot TLS Certificates on ESA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118844-technote-esa-00.html)
- [IETF RFC 3207 – SMTP Service Extension for Secure SMTP over TLS](https://datatracker.ietf.org/doc/html/rfc3207)
- [Cisco – Centralizing Services on a Secure Email and Web Manager](https://www.cisco.com/c/en/us/td/docs/security/esa/esa13-7-0/user_guide/b_ESA_Admin_Guide_13-7/b_ESA_Admin_Guide_12_1_chapter_0101011.html)
- [Cisco – Secure Email Gateway Support Documentation](https://www.cisco.com/c/en/us/support/security/email-security-appliance/series.html)
- [IANA – Service Name and Transport Protocol Port Number Registry](https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml)
- [IETF RFC 4511 – Lightweight Directory Access Protocol](https://datatracker.ietf.org/doc/html/rfc4511)
- [Cisco – Setup and Installation](https://www.cisco.com/c/en/us/td/docs/security/esa/esa13-7-0/user_guide/b_ESA_Admin_Guide_13-7/b_ESA_Admin_Guide_12_1_chapter_010.html)
- [Cisco – Resource Conservation Mode](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117834-qanda-esa-00.html)
- [Cisco – SNMP Monitoring on ESA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/117831-qanda-esa-00.html)
- [Cisco – AsyncOS User Guide: System Administration](https://www.cisco.com/c/en/us/td/docs/security/esa/esa16-0/user_guide/b_ESA_Admin_Guide_16-0/b_ESA_Admin_Guide_12_1_chapter_0100010.html)
- [Cisco – Automate Configuration Backup of a Clustered ESA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118403-technote-esa-00.html)
- [Cisco – Secure Email Gateway Release Notes](https://www.cisco.com/c/en/us/support/security/email-security-appliance/products-release-notes-list.html)
- [Cisco – Secure Email Gateway Software Lifecycle Support Statement](https://www.cisco.com/c/dam/en/us/td/docs/security/esa/lifecycle_support_statement/Secure_Email_Gateway_Software_Lifecycle_Support_Statement.pdf)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND 9 – dig manpage](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc manpage](https://man.openbsd.org/nc)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [swaks – SMTP test tool](https://jetmore.org/john/code/swaks/)
- [Microsoft Learn – Invoke-WebRequest](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/invoke-webrequest)
- [Microsoft Learn – pktmon](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – manual page](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [Cisco – Agreement to Acquire IronPort](https://newsroom.cisco.com/c/r/newsroom/en/us/a/y2007/m01/cisco-announces-agreement-to-acquire-ironport.html)
- [Cisco – Secure Email Gateway End-of-Life Notices](https://www.cisco.com/c/en/us/products/security/email-security-appliance/eos-eol-notice-listing.html)
