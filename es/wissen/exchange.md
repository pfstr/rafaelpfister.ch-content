---
title: "Microsoft Exchange: familia de productos y modelos operativos"
blatt: "exchange"
description: "Microsoft Exchange como familia de productos: conceptos y protocolos comunes, diferencias entre Exchange Online y Exchange Server, así como las funciones de la operación híbrida y el flujo de correo híbrido."
fakten:
  - label: Familia de productos
    wert: Exchange Online y Exchange Server
    href: https://learn.microsoft.com/en-us/exchange/
  - label: Funciones principales
    wert: Correo electrónico, calendario, contactos, libreta de direcciones y directivas
    href: https://learn.microsoft.com/en-us/exchange/recipients/recipients
  - label: Operación en la nube
    wert: Exchange Online dentro de Microsoft 365
    href: https://learn.microsoft.com/en-us/exchange/exchange-online
  - label: Operación propia
    wert: Exchange Server en Windows Server y Active Directory
    href: https://learn.microsoft.com/en-us/exchange/exchange-server
  - label: Coexistencia
    wert: Exchange Hybrid conecta la organización local y Exchange Online
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Transporte de correo
    wert: SMTP, conectores, reglas, colas y entrega
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow
  - label: Acceso de clientes
    wert: HTTPS, MAPI/HTTP, Outlook en la Web y clientes móviles
    href: https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access
  - label: Destinatarios
    wert: Buzones, grupos, contactos, usuarios de correo y recursos
    href: https://learn.microsoft.com/en-us/exchange/recipients/recipients
  - label: Directorios
    wert: Active Directory local, Microsoft Entra ID en la nube
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory
  - label: Administración
    wert: Exchange Admin Center, PowerShell y permisos basados en roles
    href: https://learn.microsoft.com/en-us/exchange/permissions/permissions
  - label: Diagnóstico en la nube
    wert: Message Trace, informes y Microsoft 365 Service Health
    href: https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message
  - label: Diagnóstico de servidores
    wert: Colas, Message Tracking, Health Sets y copias de bases de datos
    href: https://learn.microsoft.com/en-us/exchange/server-health/server-health
werbung:
  - tools
  - newsletter
ctaThemen:
  - microsoft-365-exchange
  - exchange-onprem-hybrid
translationSourceHash: f76fa79338367d134f158edf1b05fbee051e68cffacc657f421ca0bbdd03dc2f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:15:45.587Z
translationReview: automatic
---

# Microsoft Exchange: familia de productos y modelos operativos

El nombre **Microsoft Exchange** designa hoy dos plataformas estrechamente relacionadas, pero operadas de forma diferente. En **Exchange Online**, Microsoft opera los servidores, las copias de las bases de datos y la infraestructura interna de transporte. En **Exchange Server**, los hosts, Active Directory, las bases de datos, las colas, los certificados y la recuperación son responsabilidad de la propia organización. **Exchange Hybrid** conecta ambas organizaciones cuando los buzones o las funciones se distribuyen entre ambos lados ([Microsoft Learn: Exchange](https://learn.microsoft.com/en-us/exchange/), [Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Esta distinción es el punto de partida para cualquier otra cuestión. Un buzón puede estar en la nube o en el propio centro de datos. Sin embargo, el remitente visible, el dominio SMTP y la libreta de direcciones pueden utilizarse conjuntamente. Solo cuando se conocen la ubicación del buzón, el origen de sus atributos de destinatario y la ruta real del mensaje es posible investigar de forma útil la entrega, los permisos y los errores.

## Primero aclarar: ¿dónde está el buzón?

Exchange administra correo electrónico, calendarios, contactos, tareas, objetos de libreta de direcciones y derechos de acceso. Para los usuarios, esto es en gran medida igual. Sin embargo, para los administradores, la ubicación del buzón cambia casi todas las herramientas y responsabilidades.

| Pregunta | Exchange Online | Exchange On-Premises | Exchange Hybrid |
|---|---|---|---|
| ¿Quién opera los servidores de buzones? | Microsoft | la propia organización | ambos lados para sus respectivos buzones |
| ¿Dónde se editan los destinatarios? | Exchange Online y Entra ID | Exchange Server y Active Directory | normalmente se crean localmente y se sincronizan con Entra ID; el modelo exacto debe estar documentado |
| ¿Dónde se rastrea un mensaje? | Message Trace | Message Tracking Logs y colas | en ambos lados, vinculados mediante la hora, el remitente, el destinatario y los ID de mensaje |
| ¿Quién puede activar copias de bases de datos? | Microsoft | la propia administración de Exchange | el operador respectivo del lado afectado |
| ¿Qué conecta ambos lados? | no aplicable | no aplicable | sincronización de directorios, relaciones de organización, OAuth, Autodiscover y conectores SMTP |

La tabla es solo una visión general. Los cuatro artículos en profundidad tratan los modelos operativos por separado:

- [Exchange Online](/kb/exchange-online) explica los objetos de tenant, EOP, conectores, Message Trace, retención y la operación de un servicio en la nube.
- [Exchange On-Premises](/kb/exchange-on-premises) sigue la canalización de transporte, las bases de datos ESE, los DAG, Active Directory y la recuperación en el propio centro de datos.
- [Exchange Hybrid](/kb/exchange-hybrid) trata la sincronización de directorios, la administración de destinatarios, OAuth, las relaciones de organización, Autodiscover y el Hybrid Configuration Wizard.
- [Flujo de correo híbrido](/kb/hybrid-mailfluss) sigue los mensajes entre Internet, Exchange Online, la organización local y una puerta de enlace de correo opcional.

## Estructura técnica común: lo que permanece igual en todas las variantes de Exchange

Una vez aclarado el lugar de operación, conviene observar el modelo funcional común. Exchange conoce **destinatarios**, **mensajes**, **buzones**, **reglas de transporte**, **dominios** y **conectores**. Estos conceptos aparecen en la nube y en servidores propios, aunque los sistemas subyacentes sean accesibles de forma distinta ([Microsoft Learn: Recipients](https://learn.microsoft.com/en-us/exchange/recipients/recipients), [Mail flow and the transport pipeline](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)).

Un destinatario es, ante todo, un objeto de directorio habilitado para correo. Tiene direcciones y un tipo, como buzón de usuario, buzón compartido, grupo, contacto o usuario de correo. El objeto responde a la pregunta de **quién** representa una dirección y **adónde** debe entregar Exchange. El buzón almacena después los elementos propiamente dichos. Por ello, un objeto de destinatario defectuoso y un buzón sano pueden existir simultáneamente, o a la inversa.

La ruta de los mensajes también sigue en ambas plataformas los mismos pasos generales: Exchange acepta un mensaje SMTP, resuelve los destinatarios, aplica reglas de transporte y funciones de protección, elige el siguiente destino y entrega bien en un buzón o bien al siguiente salto SMTP. La implementación exacta difiere. En servidores propios, el administrador puede ver colas y registros de seguimiento locales; en Exchange Online están disponibles Message Trace, informes y Service Health para ello ([Exchange Server mail flow](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow), [Trace an email message in Exchange Online](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange.svg?v=20260813" title="Interaktive Infografik: Exchange-Produktfamilie mit Transport, Postfächern, Exchange Online und Hybridverbindungen" loading="lazy">
  <a href="/images/kb-interaktiv-exchange.svg?v=20260813">Abrir directamente la vista general interactiva de Exchange</a>.
</iframe>

## De la dirección a la ruta del mensaje

El dominio SMTP común suele llevar a la suposición errónea de que todos los mensajes siguen la misma ruta. En realidad, una combinación de DNS, Accepted Domains, objeto de destinatario, conectores y reglas determina adónde va un mensaje a continuación.

Un **Accepted Domain** indica a Exchange cómo debe tratarse un dominio. En el caso de un dominio autoritativo, Exchange espera todos los destinatarios válidos en su propio directorio. En el caso de un dominio de retransmisión interna, los destinatarios desconocidos pueden pasar a otro sistema. Por tanto, esta configuración no es una línea de inventario, sino parte de la decisión de entrega ([Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains), [Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)).

Los **conectores** determinan después de qué sistemas acepta Exchange mensajes y a qué sistemas los envía. En Exchange Server, los conectores de recepción y envío trabajan con enlaces locales, espacios de direcciones, servidores de origen, hosts inteligentes y permisos. Exchange Online utiliza conectores entrantes y salientes para relaciones con la propia infraestructura o con socios ([Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors), [Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)).

Solo en este punto cobran importancia las variantes híbridas. Centralized Mail Transport, una puerta de enlace previa o un dominio de destinatarios compartido modifican los saltos y las responsabilidades adicionales. Por ello, pertenecen a su propio artículo [Flujo de correo híbrido](/kb/hybrid-mailfluss), no entre los temas de identidad o clientes.

## Del inicio de sesión al buzón

La ruta del mensaje aún no explica cómo Outlook encuentra su buzón. Para ello, Exchange utiliza **Autodiscover**. Un cliente empieza con la identidad del usuario y determina a partir de ella el extremo de servicio adecuado. Localmente pueden intervenir los Service Connection Points de Active Directory y DNS; en Exchange Online, los extremos de Microsoft 365 conducen al servicio en la nube ([Microsoft Learn: Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

Tras determinar el extremo, el acceso moderno de Outlook se realiza mediante HTTPS, en particular con MAPI over HTTP. Outlook en la Web, Exchange ActiveSync y varias API también utilizan HTTPS, pero cada uno con sus propios protocolos de aplicación y permisos. Por tanto, un inicio de sesión correcto en el portal de Microsoft 365 no demuestra automáticamente que funcionen Autodiscover, el protocolo de Outlook o el acceso al buzón concreto ([Client Access protocol architecture](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access), [MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)).

En la operación híbrida se añade otra decisión: ¿el buzón está localmente o en línea? Autodiscover y los atributos de destinatario deben dirigir al cliente al lado correcto. Solo después entran en juego funciones entre organizaciones, como disponibilidad u operaciones de traslado de buzones. Este orden se trata paso a paso en el artículo [Exchange Hybrid](/kb/exchange-hybrid).

## Administración y permisos

Una vez comprendidas las rutas de datos y acceso, surge la cuestión de quién puede modificarlas. Exchange utiliza control de acceso basado en roles. Los roles contienen cmdlets y parámetros, los grupos de roles o las asignaciones de roles los conectan con los administradores, y los ámbitos limitan el alcance. Exchange Online y Exchange Server tienen conceptos de RBAC relacionados, pero configuraciones separadas ([Permissions in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions), [Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)).

En la práctica, esto significa que un rol de administrador de Entra, un grupo de roles de Exchange y un permiso de buzón no son lo mismo. **Full Access** permite abrir un buzón, **Send As** enviar como destinatario y **Send on Behalf** enviar de forma visible en nombre de otra persona. Ninguno de estos permisos explica por sí solo si una aplicación puede acceder mediante Microsoft Graph o EWS ([Manage permissions for recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)).

Por ello, la pregunta experta no es «¿Es el usuario administrador?», sino: ¿qué identidad inicia sesión, qué rol se aplica en qué organización de Exchange, qué objeto se está abordando y qué permiso adicional de buzón o aplicación se comprueba?

## La operación y la resolución de problemas comienzan por el lado correcto

Un diagnóstico útil comienza con tres datos: **usuario o destinatario afectado, hora exacta y ubicación del buzón**. Después se sigue la ruta en el orden de DNS o Autodiscover, inicio de sesión, extremo de Exchange, resolución de destinatarios, evento de transporte y entrega en el buzón.

Para Exchange Online, Message Trace y Microsoft 365 Service Health proporcionan el estado del servicio visible para el cliente. Para Exchange Server se añaden colas locales, Message Tracking Logs, Health Sets, Event Logs y copias de bases de datos. En la operación híbrida, las evidencias de ambos lados se reúnen en una línea temporal común ([Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq), [Server health and performance](https://learn.microsoft.com/en-us/exchange/server-health/server-health)).

Los artículos en profundidad contienen bloques de diagnóstico de Windows/Unix adecuados y la documentación oficial de las herramientas utilizadas. Aquí basta con la regla operativa más importante: determinar primero el lugar y la ruta, y después elegir la herramienta.

## Almacenamiento de datos, disponibilidad y recuperación

La diferencia entre la nube y la operación propia resulta más evidente en la recuperación. Exchange Server almacena buzones en bases de datos ESE con registros de transacciones. Los Database Availability Groups replican copias de bases de datos y permiten activaciones en otros servidores. No obstante, la organización sigue siendo responsable de la estrategia de copia de seguridad, la capacidad de recuperación y la dependencia de Active Directory ([Database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups), [Backup, restore and disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)).

Exchange Online opera redundancia de bases de datos y transporte como parte del servicio. En cambio, los administradores de tenant trabajan con elementos eliminados, Single Item Recovery, retención, retenciones legales y, si procede, requisitos de copia de seguridad externa. La resiliencia del servicio de Microsoft y una regla funcional de retención responden a preguntas distintas: una protege el servicio en ejecución; la otra determina qué contenidos se conservan tras la eliminación o con fines de cumplimiento normativo ([Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

En la operación híbrida, ambos modelos de recuperación deben documentarse en paralelo. Además, se necesitan sincronización de directorios, certificados, configuración de OAuth y conectores para restablecer la conexión después de una interrupción. Aunque estas configuraciones no contienen contenido de buzones, determinan si las dos organizaciones de Exchange pueden volver a colaborar.

## Desarrollo técnico

Exchange 4.0 apareció en 1996 como sucesor de las primeras plataformas de correo de Microsoft. Las primeras versiones utilizaban un directorio propio, MAPI y la familia de bases de datos ESE; los protocolos de Internet fueron adquiriendo importancia gradualmente. Con Exchange 2000, Active Directory y SMTP se convirtieron en componentes centrales. Exchange 2007 introdujo roles de servidor claramente definidos y Exchange Management Shell; Exchange 2010 introdujo el Database Availability Group ([Exchange Team: A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388), [Exchange Team: Exchange Server 2007 transport redesign](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)).

En paralelo, Microsoft desarrolló sus ofertas de Exchange hospedado hasta Exchange Online. Por ello, Hybrid no surgió como un producto individual, sino como la conexión de dos organizaciones de Exchange independientes. Exchange Server Subscription Edition continuó esta línea de desarrollo local a partir de 2025 en Modern Lifecycle. Los datos concretos de compilaciones y actualizaciones se verifican antes de realizar cambios en la documentación de Microsoft, que se mantiene continuamente ([Exchange Server SE release notes](https://learn.microsoft.com/en-us/exchange/release-notes), [Exchange Server Subscription Edition lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/exchange-server-subscription-edition)).

## Fuentes

- [Microsoft Learn – Exchange](https://learn.microsoft.com/en-us/exchange/)
- [Microsoft Learn – Exchange Online](https://learn.microsoft.com/en-us/exchange/exchange-online)
- [Microsoft Learn – Exchange Server](https://learn.microsoft.com/en-us/exchange/exchange-server)
- [Microsoft Learn – Exchange Server architecture](https://learn.microsoft.com/en-us/exchange/architecture/architecture)
- [Microsoft Learn – Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Microsoft Learn – Recipients](https://learn.microsoft.com/en-us/exchange/recipients/recipients)
- [Microsoft Learn – Mail flow and the transport pipeline](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)
- [Microsoft Learn – Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)
- [Microsoft Learn – Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)
- [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)
- [Microsoft Learn – Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)
- [Microsoft Learn – Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Microsoft Learn – Client Access protocol architecture](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)
- [Microsoft Learn – MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)
- [Microsoft Learn – Active Directory in Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)
- [Microsoft Learn – Permissions in Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions)
- [Microsoft Learn – Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)
- [Microsoft Learn – Manage permissions for recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)
- [Microsoft Learn – Server health and performance](https://learn.microsoft.com/en-us/exchange/server-health/server-health)
- [Microsoft Learn – Database availability groups](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups)
- [Microsoft Learn – Backup, restore and disaster recovery](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)
- [Microsoft Service Assurance – Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)
- [Microsoft Purview – Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)
- [Exchange Team – A brief history of time](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388)
- [Exchange Team – Exchange Server 2007 transport redesign](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)
- [Microsoft Learn – Exchange Server SE release notes](https://learn.microsoft.com/en-us/exchange/release-notes)
- [Microsoft Lifecycle – Exchange Server Subscription Edition](https://learn.microsoft.com/en-us/lifecycle/products/exchange-server-subscription-edition)
