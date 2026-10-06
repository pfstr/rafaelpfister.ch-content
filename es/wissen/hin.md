---
title: "HIN: espacio de confianza, Mail Gateway y Stargate"
blatt: "hin"
description: "HIN para administradores de mensajería y plataformas: modelo de confianza e identidad, HIN Mail, correo clásico y Access Gateway, HIN Client, PKI, rutas SMTP y de portal, almacén de correo, arquitectura Stargate, migración, monitorización, recuperación y diagnóstico."
fakten:
  - label: Plataforma
    wert: Espacio de confianza para el sector sanitario suizo
    href: https://www.hin.ch/de/services/hin-mail/hin-mail.cfm
  - label: Operador
    wert: Health Info Net AG · fundada en 1996
    href: https://www.hin.ch/de/ueber-hin/unternehmen/geschichte.cfm
  - label: Modelo de correo
    wert: HIN a HIN automáticamente · externo con marca de confidencialidad
    href: https://support.hin.ch/de/service/hin-mail-und-mobile.cfm
  - label: Edge clásico
    wert: Mail y Access como appliances virtuales
    href: https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm
  - label: Arquitectura objetivo
    wert: Postfix → MXEngine → Postfix
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
  - label: Stack principal
    wert: OPA/Rego · PostgreSQL · Vault · MinIO
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
  - label: Transporte mesh
    wert: WireGuard · IDAgent · puerto 19818
    href: https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf
  - label: Ancla de confianza
    wert: HIN Identidad · S/MIME · claves y CSR
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
  - label: Acceso web
    wert: HIN Client · Access Gateway · SAML · OAuth 2.0
    href: https://download.hin.ch/oauth2/doku/de/
  - label: Almacén de correo
    wert: separado del transporte de gateway · IMAP/POP/Webmail
    href: https://support.hin.ch/de/service/hin-gateway.cfm
  - label: Despliegue
    wert: Linux · Docker Compose · imágenes de VM
    href: https://health-info-net-ag.github.io/Stargate-deployment/de/
  - label: Observability
    wert: Prometheus · Promtail/Loki · Node Exporter
    href: https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf
werbung:
  - stargate
  - newsletter
ctaThemen:
  - hin-gateway
translationSourceHash: 4b3c48b30e1dfb939e082bba0637c401f51d48a6dc5ed8e71b90c9d8114b2c8c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-06T11:20:40.307Z
translationReview: automatic
---

# HIN: espacio de confianza, Mail Gateway y Stargate

HIN no es un único programa de cifrado, sino un espacio de confianza sectorial compuesto por identidades verificadas, servicios centrales de plataforma y componentes de acceso o gateway. **HIN Mail** protege la comunicación por correo electrónico, **HIN Access** proporciona acceso a aplicaciones web protegidas y una **HIN Identidad** vincula a la persona u organización autorizada con material criptográfico de claves. En las instituciones, los componentes de Mail y Access se sitúan en el límite entre la infraestructura propia y la plataforma HIN ([HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm), [Membresía colectiva de HIN con Gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

Para los administradores de mensajería, deben diferenciarse cuatro espacios de estado. El servidor de correo local dispone de buzones, conectores y colas. El gateway asume las decisiones de transporte, confianza y protección. La plataforma HIN proporciona servicios de directorio, identidad, claves, correo y acceso. Para destinatarios fuera de la comunidad HIN, se añade una ruta web y de autenticación. Una incidencia en un espacio no implica automáticamente una incidencia en todos los demás; por ejemplo, un puerto SMTP accesible no acredita ni una HIN Identidad válida ni una entrega correcta mediante el portal ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Mail a personas no miembros](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)).

El nombre del producto también requiere una contextualización temporal. La generación clásica de gateway incluye un Mail Gateway, MGW, y, según el contrato, un Access Gateway, AGW. Como sucesor, HIN describe el nodo mesh compatible con correo desarrollado en el proyecto **Stargate**. Sigue siendo compatible con SMTP frente a los sistemas de correo locales, pero modifica la arquitectura, la distribución de claves, el despliegue y el transporte entre organizaciones. Por ello, las afirmaciones sobre la plataforma objetivo no deben trasladarse sin comprobación a un MGW existente, ni viceversa ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

La explicación sigue un mensaje desde el remitente, pasando por la identidad HIN y el gateway, hasta el destinatario. A continuación se tratan los cambios de plataforma, las dependencias, el diagnóstico, el material de claves y la recuperación.

## Enfoque arquitectónico: espacio de confianza con componentes edge

El modelo colectivo clásico proporciona HIN Mail y HIN Access como appliances virtuales. HIN documenta el cifrado y la firma S/MIME a nivel de dominio de correo, así como un rastro de auditoría; el Access Gateway actúa como proveedor local de identidad, asigna eIDs de HIN a solicitudes verificadas y puede integrar servicios existentes de directorio y autenticación ([Membresía colectiva de HIN con Gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

Esta construcción es un **modelo de confianza edge**: la organización controla el servidor de correo, DNS, firewall, entrega interna y el recurso de gateway local; HIN opera los servicios superiores de confianza y plataforma. La finalización SMTP positiva transfiere la responsabilidad de un mensaje concreto al siguiente salto; no indica que la ruta posterior de HIN, portal o destinatario ya se haya completado. Para el nuevo stack de gateway, HIN asigna expresamente al cliente DNS y la reputación de correo ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

Stargate desplaza este límite a un nodo mesh relacionado con la organización. HIN describe una arquitectura de microservicios nativa de la nube con API REST, identidades descentralizadas, gestión distribuida de claves y políticas programables. Está prevista una instancia propia por organización; HIN cita como modalidades de despliegue imágenes virtuales y contenedores, así como entornos OpenShift y Kubernetes en la descripción del producto. Se trata de una arquitectura objetivo publicada, no de una prueba de que todas las organizaciones HIN existentes ya operen de este modo ([HIN Gateway: Arquitectura y despliegue](https://support.hin.ch/de/service/hin-gateway.cfm), [Descripción del producto HIN Gateway](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm)).

## Stack tecnológico y responsabilidades

El stack documentado públicamente consta de varias generaciones y no debe mezclarse en un único monolito:

| Capa | Implementación del nuevo gateway | Ámbito de estado y fallo | Evidencia para el administrador |
|---|---|---|---|
| Borde SMTP | **Postfix Relay** para recepción, reintentos, enrutamiento DNS y entrega | El transporte está separado de la decisión sobre el contenido | Respuesta final SMTP, ID de cola, siguiente salto, MX y PTR |
| Procesamiento | **MXEngine** para ingestión HTTP/SMTP, transformación y estrategia de entrega | Los errores de procesamiento se distinguen de los reintentos de Postfix | Message-ID, evento MXEngine, transformación, devolución a Postfix |
| Política | **Open Policy Agent con Rego**, opcionalmente sincronizado mediante Git | El estado de reglas es estado de datos y versiones | Revisión de la política, entradas, resultado, aprobación y reversión |
| Identidad y criptografía | **S/MIME Keys Client**, **IDAgent**, emisor y verificador | CSR, certificado, clave privada e identidad del par son objetos separados | Huella digital, titular, vencimiento, emisor, par y rotación |
| Persistencia | **PostgreSQL** por servicio, **Vault** para secretos, **MinIO** para mensajes y adjuntos | La base de datos, el almacén de secretos y el almacenamiento de objetos tienen límites de recuperación propios | Volumen, hora de backup, prueba de restauración y comprobación de consistencia por servicio |
| Transporte mesh | **WireGuard** mediante IDAgent | El estado del canal no equivale a la entrega SMTP | Clave de par, endpoint, handshake, puerto 19818 y evento SMTP posterior |
| Observability | **Promtail → Loki**, **Node Exporter**, métricas compatibles con Prometheus y Version Collector | El transporte de logs, las métricas del host y la salud del servicio pueden fallar por separado | Liveness, antigüedad del scrape, entrada de logs, recursos del host y referencia temporal |
| Despliegue | Linux, **Docker Compose**, volúmenes persistentes de Docker; imágenes de VM como vía de instalación | El host, los contenedores, las imágenes y los volúmenes tienen ciclos de vida diferentes | Imagen aprobada, configuración Compose, inventario de volúmenes y prueba de reinicio |

HIN documenta explícitamente el flujo de mensajes como `External SMTP → Postfix → MXEngine → Postfix → External SMTP`. La entrega forzada a MXEngine evita una elusión de políticas; Postfix sigue siendo responsable de la entrega y los reintentos. OPA/Rego mantiene las reglas de negocio fuera del código de la aplicación. PostgreSQL, Vault y MinIO almacenan clases de estado diferentes y no deben tratarse como un único sistema de archivos en el backup ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

La documentación técnica de instalación cita distribuciones Linux de familias compatibles con RHEL, así como Ubuntu y Debian, funcionamiento con Docker Compose en un único host e imágenes de VM para varias plataformas. Esta declaración de soporte depende de la versión y del despliegue; prevalece la documentación HIN aprobada en el momento de la instalación, no una lista de distribuciones copiada estáticamente ([HIN Stargate Deployment](https://health-info-net-ag.github.io/Stargate-deployment/de/)).

El almacén de correo sigue siendo otro límite: según HIN, Stargate actúa como Mail Transport Agent, mientras que IMAP permanece implementado en la plataforma Zimbra existente. Además, HIN Access constituye una ruta independiente de autenticación y autorización mediante el cliente, AGW o Access Control Service ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [Manual de HIN Client 3](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-hin.svg?v=20260813" title="Interaktive Infografik: HIN Vertrauensraum mit lokalem Mailserver, klassischem Mail und Access Gateway, HIN Identität, Mailplattform, Nichtmitglieder-Portal sowie Stargate-Zielarchitektur und Betriebssignalen" loading="lazy">
  <a href="/images/kb-interaktiv-hin.svg?v=20260813">Abrir directamente el gráfico interactivo</a>.
</iframe>

## Identidad y PKI con autorización

Una HIN Identidad es más que una dirección de correo electrónico. Durante la activación, HIN Client genera un par de claves; la contraseña desbloquea el material de claves local. En el nuevo gateway, S/MIME Keys Client genera claves criptográficas y Certificate Signing Requests, mientras que Vault alberga claves privadas, credenciales y configuración sensible. Un certificado vincula una clave pública a una identidad nombrada, pero no sustituye la autorización específica de una aplicación ([Manual de HIN Client 3](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [RFC 5280](https://datatracker.ietf.org/doc/html/rfc5280)).

El ciclo de vida debe formar parte del runbook de IAM: registro, activación, cambio de dispositivo, cambio de función o nombre, bloqueo, nuevo registro ante sospecha sobre una clave y salida. Tras el pedido, HIN exige una verificación de identidad y distingue entre eIDs personales y la ID de organización o dispositivo en el modelo colectivo. Un transporte de correo funcional no debe considerarse prueba de que una identidad anterior o asignada incorrectamente ya no tenga acceso ([HIN Identidad](https://support.hin.ch/de/service/hin-identitaet.cfm), [Membresía colectiva de HIN con Gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

HIN Access y HIN Mail comparten el espacio de confianza, pero no el mismo flujo de protocolo. Access Control Service puede solicitar un nivel de autenticación mediante la `RequestedAuthnContext` SAML; HIN documenta perfiles para contraseña y MFA. Para integraciones, HIN proporciona además flujos OAuth 2.0 para Authorization Code y Client Credentials. La autenticación, la emisión de tokens y la autorización por la aplicación de destino deben registrarse por separado ([HIN Authentication Context](https://support.hin.ch/de/thema/hin-access/pwd.cfm), [Integración OAuth2 de HIN](https://download.hin.ch/oauth2/doku/de/), [SAML 2.0 Core](https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf), [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)).

Solo cuando se han aclarado la identidad y la PKI puede evaluarse la ruta de correo. El gateway y la política deciden qué procedimiento de protección se aplica según el remitente, el destinatario y el destino.

## Rutas de correo y decisiones de protección

Con los componentes HIN implicados en mente, ya puede explicarse la ruta concreta del mensaje. Los siguientes casos se distinguen según dónde se encuentren remitente y destinatario y qué servicio toma la decisión de protección.

### Entre participantes de HIN

HIN describe los mensajes entre direcciones HIN como transmitidos automáticamente de forma compatible con la protección de datos. La plataforma indica el estado de integridad en el asunto con `[HIN secured]` o `[Not secured by HIN]`. Esta marca es una señal para el usuario, pero no una correlación técnica suficiente: para un incidente se requieren además el remitente y destinatario del sobre, Internet `Message-ID`, la cadena `Received`, el evento del gateway y la ventana temporal ([HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm), [HIN Mail y Mobile](https://support.hin.ch/de/service/hin-mail-und-mobile.cfm), [RFC 5322](https://datatracker.ietf.org/doc/html/rfc5322)).

Una ruta organizativa clásica puede describirse como `Mailserver → SMTP → MGW → HIN → MGW → SMTP → Mailserver`. Cada estación termina una sesión y puede tener su propia cola. HIN describe el modelo colectivo como cifrado y firma S/MIME a nivel de dominio de correo; la entrega local antes y después de este límite de gateway sigue siendo un ámbito propio de protección y operación ([Membresía colectiva de HIN con Gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm), [RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

### A personas sin membresía HIN

Para destinatarios sin dirección HIN, el remitente debe marcar expresamente el mensaje como confidencial según la documentación de HIN, por ejemplo con `(Vertraulich)` en el asunto. El destinatario abre el contenido protegido mediante una ruta web y se autentica con número de móvil y código SMS; es posible una respuesta segura. Esto crea estados adicionales: correo de notificación, objeto del portal, registro del destinatario, segundo factor, conservación y canal de respuesta ([HIN Mail a personas no miembros](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm), [HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)).

Una notificación entregada aún no es un mensaje leído. Por ello, las pruebas sintéticas deberían cubrir toda la ruta hasta el inicio de sesión, la apertura, el adjunto y la respuesta. Los reenvíos desde el portal pueden abandonar la ruta de protección; HIN señala expresamente que determinados tipos de reenvío se envían sin cifrar. Estas acciones de usuario deben incluirse en la formación, el modelo DLP y el análisis de incidentes ([HIN Mail a personas no miembros](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)).

### Dispositivos, aplicaciones y envíos masivos

El Mail Gateway clásico puede servir como interfaz SMTP para dispositivos internos; HIN también documenta esta capacidad para Stargate. La página de servicios HIN Mail indica que el envío desde sistemas a través del gateway requiere una licencia independiente. Por tanto, escáneres, HIS, aplicaciones de laboratorio y procesos por lotes necesitan una ruta propia y documentada de remitente, relay, volumen y errores, en lugar de usar silenciosamente el flujo de correo de los usuarios ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)).

## Enrutamiento SMTP y límites de aceptación

El gateway debe saber con precisión para cada dirección qué dominios son locales, autoritativos, deben reenviarse o deben rechazarse. Una responsabilidad poco clara entre Exchange, conector de nube, Secure Mail Gateway y edge de HIN genera elusiones o bucles. [SMTP](/kb/smtp) prescribe líneas de trazado y describe la detección de bucles; la transferencia real de responsabilidad solo ocurre con una respuesta positiva tras el contenido completo del mensaje ([RFC 5321, información de trazado](https://datatracker.ietf.org/doc/html/rfc5321#section-4.4), [RFC 5321, DATA](https://datatracker.ietf.org/doc/html/rfc5321#section-4.1.1.4)).

Para cada ruta, la documentación operativa debe incluir como mínimo estos valores:

- listener local, puerto y redes de origen o identidades de par esperadas;
- nombre EHLO, dominios de sobre y ámbitos de relay autorizados;
- orden de filtro antispam/antimalware, gateway HIN y sistema de correo interno;
- siguiente salto, resolución DNS o de smarthost y requisito de [TLS](/kb/tls);
- comportamiento cuando la plataforma HIN, el componente de identidad o de claves no es accesible;
- antigüedad de la cola, plan de reintentos, número máximo de saltos y responsabilidad de los rebotes;
- rutas de excepción para dispositivos, envío de sistemas y coexistencia durante la migración.

Para Exchange Online, se trata de una cuestión de conectores y no solo de DNS. Microsoft documenta el enrutamiento hacia gateways de terceros y las condiciones de conectores basadas en certificado o IP. Las preguntas frecuentes de HIN confirman la compatibilidad básica con arquitecturas híbridas y cercanas a Microsoft 365, pero remiten a documentación de migración específica del cliente para la configuración concreta ([Microsoft: flujo de correo con conectores](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow), [Microsoft: flujo de correo de terceros en la nube](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud), [HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

## Almacén de correo y acceso de clientes con token

El acceso al buzón y el transporte de gateway son modelos operativos diferentes. HIN publica para cuentas personales HIN Mail IMAP en el puerto 993 con TLS y Message Submission en el puerto 587 con STARTTLS; como contraseña se utiliza un token de correo generado. POP en el puerto 995 también está documentado, pero normalmente descarga los mensajes localmente y puede eliminarlos del servidor. Las funciones de protocolo corresponden a [IMAP](/kb/apache-james#protokolle-tls-und-ports), POP3 y Message Submission, no al transporte de gateway a gateway ([HIN Mail Token Service](https://support.hin.ch/de/service/hin-mail-und-mobile/token-service-mail-client.cfm), [Configuración POP de HIN](https://support.hin.ch/de/service/hin-mail-und-mobile/mail-clients-einrichten-pop.cfm), [RFC 9051](https://datatracker.ietf.org/doc/html/rfc9051), [RFC 1939](https://datatracker.ietf.org/doc/html/rfc1939), [RFC 6409](https://datatracker.ietf.org/doc/html/rfc6409)).

El manual de HIN Client documenta la sustitución del proxy de correo local histórico por acceso basado en tokens. HIN señala expresamente para servidores de terminal que, con el proxy desactivado, el envío y la recepción se realizan mediante tokens de correo. Esto es relevante para la historia técnica: originalmente, el cliente era un intermediario de comunicación para HTTP, SMTP, POP e IMAP; los modelos operativos posteriores separan más el acceso del cliente de correo y el cliente de identidad HIN ([Manual de HIN Client 3](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf), [HIN Client en servidores de terminal](https://support.hin.ch/de/thema/hin-client/hin-client-auf-terminalserver.cfm)).

En las preguntas frecuentes de Stargate, HIN describe el Mail Storage Agent existente como basado en Zimbra y, por el momento, no afectado por el nuevo transporte de gateway. Por tanto, un flujo de correo Stargate satisfactorio no demuestra ni la disponibilidad de IMAP ni la consistencia del buzón. A la inversa, Webmail puede funcionar mientras el conector de la organización o el transporte mesh están afectados ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

La ruta clásica de almacén de correo no es la única modalidad operativa. Stargate desplaza funciones y responsabilidades y, por tanto, debe entenderse como una ruta independiente de mensajería y administración.

## Stargate como arquitectura objetivo

Stargate pretende conservar la compatibilidad del correo en el borde y, al mismo tiempo, permitir un intercambio de datos descentralizado más general. HIN menciona Self-Sovereign Identity, Data Mesh, microservicios, API RESTful, componentes de código abierto y un nodo mesh compatible con correo. Para el canal entre instancias Stargate se anuncia un transporte aplicación a aplicación basado en [WireGuard](https://www.wireguard.com/protocol/); HIN lo diferencia expresamente de un túnel VPN general ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

Para los administradores, de ello se deriva un trazado tripartito:

1. A nivel local, SMTP con respuesta de aceptación, cola y conector sigue siendo el límite verificable.
2. Entre los nodos mesh se añaden estados de identidad, descubrimiento, claves y canal WireGuard.
3. En el destino vuelve a surgir una ruta SMTP local hacia el sistema de correo receptor.

Las vistas generales del sistema publicadas acreditan Postfix, MXEngine, OPA/Rego, PostgreSQL, Vault, MinIO, IDAgent, Promtail/Loki y métricas compatibles con Prometheus. No se especifica allí un lenguaje de programación ni un árbol fuente completo de los servicios de producto; por ello, esta información no se deduce de imágenes de contenedores ni de proyectos ajenos con el mismo nombre ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

Los límites de red del nuevo gateway están documentados públicamente con un nivel de detalle inusual:

| Puerto y dirección | Función | Importancia operativa |
|---|---|---|
| TCP 25 entrante y saliente | Aceptación SMTP y entrega basada en MX | Exposición a Internet, reputación, cola y enrutamiento de siguiente salto |
| TCP 8084 entrante | Callback HTTP del Sealer remoto | Según HIN, intencionadamente sin capa TLS adicional porque la carga útil está cifrada |
| TCP y UDP 19818 en ambas direcciones | WireGuard entre IDAgents | Comprobar conjuntamente clave de par, endpoint, NAT y firewall |
| TCP 443 y 4433 saliente | Registro, CA S/MIME, Sealer, Issuer, logging y Verifier | Dependencia de la plataforma pese al funcionamiento local del gateway |
| TCP y UDP 53 saliente | Resolución MX, SPF, A/AAAA y PTR | El enrutamiento y las decisiones de seguridad dependen de [DNS](/kb/dns) |

Los puertos de diagnóstico y servicio expuestos localmente, incluidos PostgreSQL, Vault, MinIO y los endpoints de métricas, no deberían ser accesibles desde Internet público según HIN. Los accesos administrativos a 22, 443, 8180 o 8190 deben pertenecer a una red de gestión definida y no a una autorización genérica de Internet ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf), [HIN Stargate Deployment](https://health-info-net-ag.github.io/Stargate-deployment/de/)).

HIN describe la migración como una implantación paralela del nuevo gateway. Solo tras la configuración, las pruebas y la confirmación de la disponibilidad operativa, el cliente decide la conmutación. En principio, las direcciones IP existentes pueden reutilizarse, pero la configuración debe traducirse. Por tanto, un runbook de cutover seguro incluye al menos inventario, exportación, ruta paralela, matriz de pruebas, criterio de conmutación, ruta de retorno, tratamiento de colas y rollback claro ([HIN Gateway: Migración](https://support.hin.ch/de/service/hin-gateway.cfm)).

## Alta disponibilidad con backup y recuperación

HIN documenta un clúster OpenShift redundante para su propia parte de Stargate, pero no planifica automáticamente dos máquinas virtuales redundantes del lado del cliente. Estas afirmaciones no deben combinarse para inferir una alta disponibilidad general de extremo a extremo. Un servicio de plataforma redundante no protege frente a un único hipervisor local, una regla de conector errónea, una clave vencida o un firewall bloqueado ([HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm)).

En el nuevo gateway, los servicios persisten mediante volúmenes Docker. Vault se sella automáticamente después de reiniciar un contenedor y requiere un proceso de unseal controlado. La vista técnica indica por defecto backups diarios de bases de datos, claves y secretos de Vault, archivos de configuración y certificados; la responsabilidad y la conservación deben acordarse con el cliente ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

Un inventario de recuperación debería contener al menos, por generación:

| Objeto | Efecto de la pérdida | Paso de recuperación verificable |
|---|---|---|
| Host, definición Compose y de contenedores | El nodo edge no inicia | Preparar un nuevo destino a partir de una imagen aprobada y generar un runtime determinista |
| Configuración del cliente y políticas OPA/Rego | Ruta, política o dominio incorrectos | Aplicar estado versionado, cargar la política y ejecutar la matriz de pruebas |
| Bases de datos PostgreSQL | Falta el estado de políticas, metadatos o agentes | Restaurar la base de datos por servicio y realizar una comprobación referencial |
| Claves, secretos y certificados de Vault | Falta descifrado, identidad del par o de la organización | Realizar restauración aprobada, unseal y prueba de función criptográfica |
| Mensajes y adjuntos MinIO | Falta un mensaje u objeto de archivo | Aclarar por separado alcance y conservación con HIN y probar la restauración de objetos |
| Conectores y DNS | Elusión, bucle o imposibilidad de entrega | Comprobar la ruta en ambas direcciones con un Message-ID inequívoco |
| Cola o prueba de transferencia | Duplicados o pérdida de mensajes | Aclarar la responsabilidad pendiente por mensaje antes de la conmutación |
| Buzón y token de correo | Acceso del cliente interrumpido | Validar por separado mediante Webmail e IMAP/Submission |
| Logs de auditoría y operativos | No es posible reconstruir el incidente | Probar referencia temporal, exportación, conservación y entrada SIEM |

La lista pública de backups menciona base de datos, Vault, configuración y certificados, pero no menciona explícitamente mensajes y adjuntos MinIO. De ello no puede deducirse ni que se aseguren ni que se excluyan intencionadamente; precisamente este punto debe constar por escrito en el acuerdo de retención, backup y restauración antes de la puesta en producción. Además, una instantánea de VM genérica no acredita ni un estado PostgreSQL consistente ni un Vault recuperable ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)).

En caso de incidencia, el mensaje se sigue desde la aceptación, pasando por la decisión de identidad y política, hasta la ruta de entrega elegida; solo después se reinician o evitan componentes individuales.

## Monitorización y triaje de incidentes

Una visión operativa útil combina señales locales y centrales:

- disponibilidad y antigüedad de cola por siguiente salto SMTP;
- tasas de aceptación, transferencia y rebote con Message-ID correlacionable;
- errores de HIN, portal, identidad, claves y políticas por separado;
- vencimiento de certificados, estado de registro y token;
- acceso al buzón mediante Webmail e IMAP independientemente del gateway;
- resolución DNS y de firewall mediante nombres en lugar de direcciones IP de HIN codificadas de forma fija;
- avisos de plataforma en [HIN Status](https://status.hin.ch/) y telemetría local.

El nuevo stack proporciona métricas de servicio compatibles con Prometheus, métricas de host mediante Node Exporter, envío central de logs mediante Promtail a Loki y un Version Collector que consulta endpoints de liveness. Para las alertas, deben tratarse por separado como mínimo la ausencia de scrape, la falta de entrada de logs, la antigüedad de cola de Postfix, los errores de MXEngine, el estado sellado de Vault, la capacidad de PostgreSQL y MinIO, el estado de pares WireGuard y el vencimiento de certificados ([HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf), [HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)).

La documentación de firewall de HIN recomienda nombres DNS, ya que las direcciones IP pueden cambiar, y cita rutas HTTPS y SMTP, entre otras, para servicios de cliente. Una página de estado global no puede detectar una incidencia local de DNS, NAT, MTU, conector o claves. Por ello, el triaje comienza por el alcance: un usuario, una identidad, un dominio, una dirección, un gateway o la plataforma ([Ajustes de firewall de HIN](https://support.hin.ch/de/thema/hin-client/anpassung-firewall-hin-client.cfm), [HIN Status](https://status.hin.ch/)).

La oferta colectiva clásica cita un rastro de auditoría para el flujo de correo; el nuevo gateway añade logs centrales estructurados. Para un análisis integral de correo, estas evidencias deben correlacionarse con logs SMTP locales y del sistema de correo. La sincronización horaria y las zonas horarias uniformes son requisitos operativos, no ajustes estéticos ([Membresía colectiva de HIN con Gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)).

## Herramientas de diagnóstico

El diagnóstico comienza por el nombre público o interno y después sigue la ruta real del correo. Solo cuando DNS, conexión y certificado son correctos se evalúan el estado del gateway, la cola y los eventos específicos de HIN.

### DNS y endpoints de HIN

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für HIN-DNS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName gateway.hin.ch -Type A
Resolve-DnsName gateway.hin.ch -Type AAAA
Resolve-DnsName smtp.mail.hin.ch -Type A
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig +short A gateway.hin.ch
dig +short AAAA gateway.hin.ch
dig +short A smtp.mail.hin.ch
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) y [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) muestran si los nombres documentados pueden resolverse desde la perspectiva del resolver realmente utilizada. Esto es más importante que un valor IP copiado, especialmente con Split-DNS y proxies; HIN recomienda expresamente nombres DNS en lugar de direcciones fijadas a largo plazo ([Ajustes de firewall de HIN](https://support.hin.ch/de/thema/hin-client/anpassung-firewall-hin-client.cfm)).

### Accesibilidad TCP y TLS

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für HIN-TCP- und TLS-Diagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Test-NetConnection hin-gateway.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection hin-gateway.example.ch -Port 19818 -InformationLevel Detailed
curl.exe --verbose --ssl-reqd smtp://hin-gateway.example.ch:25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
nc -vz hin-gateway.example.ch 25
nc -vz hin-gateway.example.ch 19818
nc -vzu hin-gateway.example.ch 19818
openssl s_client -starttls smtp -connect hin-gateway.example.ch:25 \
  -servername hin-gateway.example.ch -showcerts
```

  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) acreditan la ruta TCP. La llamada UDP de `nc` puede, como mucho, indicar accesibilidad; dado que WireGuard descarta paquetes no autorizados sin responder, solo el handshake autenticado del par constituye una prueba sólida para el puerto 19818. [`curl.exe`](https://curl.se/docs/manpage.html) de Windows y [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) comprueban el borde SMTP/STARTTLS. Un handshake correcto aún no demuestra el procesamiento de políticas ni la entrega del mensaje ([HIN Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf), [RFC 8446](https://datatracker.ietf.org/doc/html/rfc8446)).

### Transacción SMTP controlada

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für einen autorisierten HIN-Gateway-SMTP-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
curl.exe --verbose --url smtp://hin-gateway.intern.example:25 `
  --mail-from hin-test@example.ch `
  --mail-rcpt test-recipient@example.net `
  --upload-file .\hin-test.eml
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
swaks --server hin-gateway.intern.example --port 25 \
  --from hin-test@example.ch --to test-recipient@example.net \
  --data hin-test.eml
```

  </div>
</div>

[`curl`](https://curl.se/docs/manpage.html) y [`swaks`](https://jetmore.org/john/code/swaks/) solo deben utilizarse contra un listener y destinatario de prueba expresamente autorizados. Se registran la respuesta final tras `DATA`, el ID de cola local, el evento del gateway, la ruta de protección elegida, el siguiente salto y la llegada real. Un `250` en `RCPT TO` aún no supone aceptación del contenido del mensaje ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

### Estados de sockets locales

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für lokale HIN-Gateway-Socketdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Get-NetTCPConnection -State Listen,Established |
  Where-Object LocalPort -In 25,443,587,993
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
ss -tanp '( sport = :25 or sport = :443 or sport = :587 or sport = :993 )'
```

  </div>
</div>

[`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection) y [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) muestran listeners locales y sesiones TCP establecidas. Solo tienen sentido donde el administrador dispone de acceso al host correspondiente; un producto gestionado de appliance o contenedor no debe modificarse mediante accesos de shell no documentados.

### Ruta de paquetes en el punto de medición adecuado

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für HIN-Paketerfassung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
pktmon filter remove
pktmon filter add HIN-SMTP -p 25
pktmon start --capture --pkt-size 0 --file-name hin.etl
pktmon stop
pktmon pcapng hin.etl -o hin.pcapng
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
tcpdump -ni any -s 0 -w hin.pcap \
  'tcp port 25 or tcp port 443 or tcp port 587 or tcp port 993'
```

  </div>
</div>

[`pktmon`](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon) y [`tcpdump`](https://www.tcpdump.org/manpages/tcpdump.1.html) solo ven el tráfico en el punto de medición elegido. Para Stargate, un trazado SMTP local no puede explicar por completo el canal mesh; para ello se requieren adicionalmente eventos de gateway y plataforma. Las capturas pueden contener direcciones, líneas de asunto o partes de protocolo sin cifrar y deben tratarse como datos operativos sensibles.

## Historia técnica

La FMH y la Ärztekasse fundaron Health Info Net AG en 1996, cuando surgió el correo electrónico en el sector sanitario y se reconoció que el envío de datos sensibles mediante correo de Internet convencional estaba insuficientemente protegido. HIN comenzó así como proveedor de comunicación segura por correo electrónico para el colectivo médico y evolucionó hacia un espacio más amplio de confianza y acceso ([Historia de la empresa HIN](https://www.hin.ch/de/ueber-hin/unternehmen/geschichte.cfm)).

La arquitectura del cliente muestra el cambio técnico. HIN Client 1 y 2 fueron sustituidos por HIN Client 3. Por motivos de compatibilidad, el cliente siguió funcionando como proxy local para navegadores y programas de correo; para el acceso web, HIN documentó posteriormente Challenge/Response y, para las cuentas de correo, la transición a tokens independientes mediante puertos estándar. La historia explica por qué las guías de instalación antiguas indican puertos proxy locales, mientras que los documentos más recientes utilizan endpoints directos de IMAP, POP y Submission ([Manual de HIN Client 3](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf), [HIN Mail Token Service](https://support.hin.ch/de/service/hin-mail-und-mobile/token-service-mail-client.cfm)).

El modelo colectivo clásico de HIN agrupaba appliances de Mail y Access en la red del cliente. La página de servicios actual documenta para esta generación S/MIME a nivel de dominio de correo, rastro de auditoría, un proveedor de identidad local y la conexión de servicios existentes de autenticación y directorio. Estas funciones explican la separación histórica entre transporte de correo y acceso web ([Membresía colectiva de HIN con Gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)).

A partir de 2025, HIN introdujo una nueva entrega a personas no miembros; al mismo tiempo, se renovaron la infraestructura de plataforma y Access. La documentación de Gateway publicada en 2026 describe Stargate como el siguiente cambio generacional: de un gateway exclusivamente de cifrado de correo a un nodo descentralizado y nativo de la nube para correo e intercambio estructurado de datos sanitarios. La documentación técnica concreta este cambio con Postfix, MXEngine, OPA/Rego, PostgreSQL, Vault, MinIO y operación en contenedores. Por tanto, para un plan de migración no basta con sustituir una VM, sino que se requiere una nueva evaluación de identidad, claves, transporte, observability y recuperación ([HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm), [HIN Access](https://support.hin.ch/de/thema/hin-access.cfm), [HIN Gateway](https://support.hin.ch/de/service/hin-gateway.cfm), [HIN Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)).

## Fuentes

- [HIN – HIN Mail](https://www.hin.ch/de/services/hin-mail/hin-mail.cfm)
- [HIN – Membresía colectiva con Gateway](https://www.hin.ch/de/hin-mitgliedschaft/kollektivmitgliedschaft/mit-gateway.cfm)
- [Soporte HIN – HIN Gateway y Stargate](https://support.hin.ch/de/service/hin-gateway.cfm)
- [Soporte HIN – HIN Mail a personas no miembros](https://support.hin.ch/de/service/hin-mail-an-nichtmitglieder.cfm)
- [HIN – Gateway Top-Level System Overview](https://www.hin.ch/files/pdf1/hin-gateway-top-level-system-overview.pdf)
- [RFC 5321 – Simple Mail Transfer Protocol](https://datatracker.ietf.org/doc/html/rfc5321)
- [HIN – Descripción del producto HIN Gateway](https://www.hin.ch/de/services/hin-mail/hin-gateway.cfm)
- [HIN – Gateway Technical and Operational Overview](https://www.hin.ch/files/pdf1/hin-gateway-high-level-technical--operational-overview-v3.pdf)
- [HIN – Stargate Deployment](https://health-info-net-ag.github.io/Stargate-deployment/de/)
- [HIN – Manual de HIN Client 3](https://download.hin.ch/documentation/HIN_Client3_Handbuch_de_v1.1.pdf)
- [RFC 5280 – Internet X.509 PKI](https://datatracker.ietf.org/doc/html/rfc5280)
- [Soporte HIN – HIN Identidad](https://support.hin.ch/de/service/hin-identitaet.cfm)
- [Soporte HIN – SAML Authentication Context](https://support.hin.ch/de/thema/hin-access/pwd.cfm)
- [HIN – Integración OAuth2](https://download.hin.ch/oauth2/doku/de/)
- [OASIS – SAML 2.0 Core](https://docs.oasis-open.org/security/saml/v2.0/saml-core-2.0-os.pdf)
- [RFC 6749 – OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749)
- [Soporte HIN – HIN Mail y Mobile](https://support.hin.ch/de/service/hin-mail-und-mobile.cfm)
- [RFC 5322 – Internet Message Format](https://datatracker.ietf.org/doc/html/rfc5322)
- [Microsoft – Flujo de correo con conectores](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow)
- [Microsoft – Flujo de correo mediante un servicio cloud de terceros](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)
- [Soporte HIN – HIN Mail Token Service](https://support.hin.ch/de/service/hin-mail-und-mobile/token-service-mail-client.cfm)
- [Soporte HIN – Configuración POP](https://support.hin.ch/de/service/hin-mail-und-mobile/mail-clients-einrichten-pop.cfm)
- [RFC 9051 – IMAP4rev2](https://datatracker.ietf.org/doc/html/rfc9051)
- [RFC 1939 – POP3](https://datatracker.ietf.org/doc/html/rfc1939)
- [RFC 6409 – Message Submission](https://datatracker.ietf.org/doc/html/rfc6409)
- [Soporte HIN – HIN Client en servidores de terminal](https://support.hin.ch/de/thema/hin-client/hin-client-auf-terminalserver.cfm)
- [WireGuard – Protocol and Cryptography](https://www.wireguard.com/protocol/)
- [HIN Status](https://status.hin.ch/)
- [Soporte HIN – Ajustes de firewall para HIN Client](https://support.hin.ch/de/thema/hin-client/anpassung-firewall-hin-client.cfm)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [ISC BIND – Manual de dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – Manual de nc](https://man.openbsd.org/nc)
- [curl – Manual de línea de comandos](https://curl.se/docs/manpage.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [RFC 8446 – TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446)
- [swaks – Swiss Army Knife for SMTP](https://jetmore.org/john/code/swaks/)
- [Microsoft – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Microsoft – Packet Monitor](https://learn.microsoft.com/en-us/windows-server/networking/technologies/pktmon/pktmon)
- [tcpdump – tcpdump(1)](https://www.tcpdump.org/manpages/tcpdump.1.html)
- [HIN – Historia de la empresa](https://www.hin.ch/de/ueber-hin/unternehmen/geschichte.cfm)
- [Soporte HIN – HIN Access](https://support.hin.ch/de/thema/hin-access.cfm)
