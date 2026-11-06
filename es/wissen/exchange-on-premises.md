---
title: "Exchange On-Premises: arquitectura y operación del servidor"
blatt: "exchange-on-premises"
description: "Exchange en el propio centro de datos: roles de buzón y Edge, canalización de transporte, Active Directory, bases de datos ESE, DAG, acceso de clientes, seguridad, monitorización, copia de seguridad y recuperación."
fakten:
  - label: Función del producto
    wert: Plataforma de mensajería y colaboración operada de forma autónoma
    href: https://learn.microsoft.com/en-us/exchange/exchange-server
  - label: Roles de servidor
    wert: Buzón y Edge Transport opcional
    href: https://learn.microsoft.com/en-us/exchange/architecture/architecture
  - label: Sistema operativo
    wert: Windows Server conforme a los requisitos del sistema de Exchange
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements
  - label: Directorio
    wert: Active Directory Domain Services
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory
  - label: Almacenamiento de buzones
    wert: Base de datos ESE, registros de transacciones y punto de control
    href: https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange
  - label: Alta disponibilidad
    wert: Database Availability Group y copias de bases de datos
    href: https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups
  - label: Transporte
    wert: Frontend Transport, Transport Service y Mailbox Transport
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow
  - label: Resiliencia del transporte
    wert: Shadow Redundancy y Safety Net
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability
  - label: Acceso de clientes
    wert: HTTPS, MAPI/HTTP, Outlook en la Web, EWS y ActiveSync
    href: https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access
  - label: Administración
    wert: Exchange Admin Center y Exchange Management Shell
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/admin-interface
  - label: Monitorización
    wert: Managed Availability, conjuntos de estado, registros de eventos e indicadores de rendimiento
    href: https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability
  - label: Recuperación
    wert: Server Recovery, restauración de bases de datos y Recovery Database
    href: https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - smtp-mailflow
translationSourceHash: 45315e2f767bcdfb1b554eea6df80335a5ca86bbc102a345b83dd0e54cb0693b
translationModel: gpt-5.6-terra
translatedAt: 2026-11-06T03:34:33.781Z
translationReview: automatic
---

# Exchange On-Premises: arquitectura y operación del servidor

**Exchange On-Premises** significa que la organización opera Exchange en su propia infraestructura. Controla los hosts de Windows, Active Directory, los certificados, los servicios de transporte, las colas, las bases de datos de buzones y la recuperación. Microsoft proporciona el código del producto, la documentación y las actualizaciones; la disponibilidad y el mantenimiento seguro siguen siendo responsabilidad del operador ([Documentación de Exchange Server](https://learn.microsoft.com/en-us/exchange/exchange-server), [Arquitectura de Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/architecture)).

La diferencia práctica respecto a Exchange Online se aprecia de inmediato ante una incidencia. Un administrador local puede examinar una cola de transporte en un servidor concreto, comprobar el estado de una copia de base de datos y cambiar de forma controlada a otra copia. Sin embargo, también debe comprender cómo interactúan SMTP, Active Directory, ESE, Windows Failover Clustering, IIS y los servicios de Exchange.

## El servidor de buzones es el componente central

Los servidores Exchange modernos utilizan el **servidor de buzones** como componente común. Contiene servicios de acceso de clientes que aceptan y reenvían conexiones, servicios de transporte para el flujo de mensajes y el Information Store con las bases de datos de buzones. Una instalación puede empezar siendo pequeña; varios servidores y copias de bases de datos amplían el mismo modelo básico para proporcionar alta disponibilidad ([Arquitectura de Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/architecture)).

Esta consolidación no significa que todas las funciones tengan el mismo estado. Un frontend HTTPS puede estar accesible aunque la base de datos solicitada no esté montada. SMTP puede aceptar conexiones mientras un mensaje queda posteriormente esperando en una cola. Por ello, el diagnóstico sigue el recorrido real y no solo el estado global del servidor.

El **rol de transporte perimetral** opcional suele ubicarse en la red perimetral y procesa exclusivamente tráfico SMTP. EdgeSync transfiere información seleccionada de destinatarios y configuración a una instancia local de AD LDS. Edge no mantiene ninguna base de datos de buzones ni sustituye a los servidores de buzones internos ([Servidores de Edge Transport](https://learn.microsoft.com/en-us/exchange/architecture/edge-transport-servers/edge-transport-servers)).

## Pila tecnológica y dependencias

El componente de servidor determina la pila tecnológica. Exchange se ejecuta en versiones compatibles de Windows Server y utiliza Active Directory para la configuración de la organización, los servidores y los destinatarios. IIS proporciona puntos de conexión HTTP. PowerShell constituye la interfaz de administración. ESE almacena datos de buzones y colas en bases de datos independientes ([Requisitos del sistema de Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements), [Active Directory en Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)).

| Tecnología | Función en la operación de Exchange | Pregunta administrativa importante |
|---|---|---|
| Windows Server | Procesos, servicios, red, almacén de certificados y registros de eventos | ¿El host está en buen estado y correctamente actualizado? |
| Active Directory | Organización de Exchange, servidores, destinatarios, RBAC e información de enrutamiento | ¿El cambio correcto es visible en los controladores de dominio utilizados? |
| IIS y HTTPS | Outlook en la Web, EAC, EWS, ActiveSync, Autodiscover y frontends MAPI/HTTP | ¿Coinciden el nombre, el certificado, la autenticación y la ruta de backend? |
| SMTP y TLS | Aceptación y reenvío de mensajes | ¿Qué conector aceptó el mensaje y qué siguiente salto se eligió? |
| ESE | Bases de datos de buzones, cola de transporte y registros de transacciones | ¿Qué base de datos y secuencia de registros corresponden entre sí? |
| PowerShell | Administración mediante cmdlets y RBAC | ¿Qué rol, ámbito y contexto de servidor se aplican? |

Para los expertos, la dependencia de Active Directory es especialmente importante. La instalación de Exchange amplía el esquema y escribe la configuración de la organización en la partición de configuración. Los atributos de destinatario se encuentran en la partición de dominio. Por tanto, un retraso de replicación o un controlador de dominio inaccesible puede afectar de forma diferente a distintas funciones.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange-onprem.svg?v=20260813" title="Interaktive Infografik: Exchange-On-Premises-Pfad von Client und SMTP über Mailboxserver, Transport, Active Directory, ESE und DAG" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-onprem.svg?v=20260813">Abrir directamente el gráfico interactivo de Exchange On-Premises</a>.
</iframe>

## La canalización de transporte paso a paso

Con la base técnica establecida, se puede interpretar con más detalle el recorrido de los mensajes. Una conexión SMTP entrante llega primero al Front End Transport Service. Este acepta el diálogo y lo intermedia hacia el Transport Service; no entrega el mensaje directamente a un buzón ([Flujo de correo y canalización de transporte](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)).

El **Transport Service** almacena el mensaje en su base de datos de colas. A continuación lo categoriza: se resuelven los destinatarios, se ejecutan reglas y agentes de transporte, y el enrutamiento determina el siguiente salto. Para un buzón local, Mailbox Transport Delivery entrega el mensaje al Store. Un mensaje enviado desde el buzón vuelve al transporte mediante Mailbox Transport Submission.

Este orden explica observaciones habituales. Una prueba SMTP satisfactoria solo demuestra la aceptación en el frontend. Un evento `RECEIVE` en el seguimiento de mensajes aún no prueba la entrega. Solo los eventos posteriores, la cola y, si procede, el estado del Store muestran dónde terminó el proceso ([Seguimiento de mensajes](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)).

Los agentes de transporte y las reglas de flujo de correo pueden rechazar, redirigir, copiar o modificar mensajes. Dado que pueden crearse varias instancias de transporte, la búsqueda no debería realizarse únicamente por asunto. El identificador de mensaje de red, el identificador de mensaje de Internet, el remitente, el destinatario, la hora y el servidor conforman conjuntamente un rastro más fiable.

## Enrutamiento, dominios y conectores

Tras la aceptación, Exchange debe saber si un destinatario es local o si el mensaje debe reenviarse. Los **dominios aceptados** describen esta relación. Un dominio autoritativo espera que todos los destinatarios válidos pertenezcan a la propia organización. Un dominio de retransmisión interna permite reenviar destinatarios desconocidos. External Relay entrega el dominio por completo a otro servidor de correo ([Dominios aceptados en Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)).

Los conectores de recepción clasifican las sesiones entrantes según el enlace local, el intervalo de IP remotas, la autenticación y los permisos. Los conectores de envío seleccionan una ruta saliente a partir del espacio de direcciones, el coste, los servidores de origen y el enrutamiento DNS o mediante host inteligente. Varios conectores adecuados se evalúan según las reglas de enrutamiento documentadas; el nombre de un conector no controla la selección ([Conectores en servidores Exchange](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors), [Enrutamiento de correo en Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing)).

Para la operación normal basta un modelo sencillo: el conector de recepción explica **cómo entra un mensaje**; el dominio aceptado y la resolución de destinatarios explican **si Exchange es responsable**; el conector de envío y el enrutamiento explican **adónde continúa**. Los expertos añaden sitios de AD, grupos de entrega, pertenencia a DAG, ámbito de conectores y reglas de transporte.

## Base de datos de buzones, registros y punto de control

Cuando el transporte entrega al Store, comienza otra parte del sistema. Exchange almacena los buzones en bases de datos de buzones ESE. Los cambios se escriben primero en los registros de transacciones y posteriormente se incorporan al archivo `.edb`. El archivo de punto de control registra hasta qué posición de registro se han escrito las páginas de la base de datos ([Registros de transacciones y archivos de punto de control](https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange)).

Este orden permite la recuperación tras un bloqueo, pero exige archivos relacionados entre sí. Un archivo `.edb` copiado sin los registros correspondientes y sin un estado de apagado conocido no se puede restaurar automáticamente. Asimismo, una copia de seguridad no debe eliminar de forma incontrolada archivos de registro que aún se necesiten para la recuperación o la replicación.

La cola de transporte también utiliza ESE, pero es una base de datos independiente con sus propios registros. Por ello, la base de datos de buzones y la cola se supervisan y restauran por separado. Una base de datos de buzones sana no elimina un siguiente salto SMTP bloqueado; una cola vacía no repara una copia de buzón dañada ([Colas y la base de datos de colas](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)).

## Database Availability Group y Active Manager

Un único servidor de buzones explica la operación normal. Para alta disponibilidad, varios servidores se unen en un **Database Availability Group**, DAG. Cada base de datos de buzones tiene exactamente una copia activa y puede tener copias pasivas en otros miembros del DAG. Los cambios se transfieren mediante replicación de registros y bloques, y se reproducen en las copias pasivas ([Grupos de disponibilidad de bases de datos](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups), [Copias de bases de datos de buzones](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-copies)).

El **Active Manager** del Microsoft Exchange Replication Service decide qué copia está activa. Best Copy and Server Selection evalúa, entre otros factores, el estado de copia y reproducción, los bloqueos de activación y el estado del servidor. Por ello, una cola de copia en cero es útil, pero no demuestra por completo que una copia pueda activarse de inmediato ([Active Manager](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/active-manager)).

La alta disponibilidad del transporte protege otra sección del recorrido. Shadow Redundancy conserva una copia adicional mientras el mensaje está en tránsito. Safety Net mantiene mensajes ya procesados para una posible reentrega tras la activación de una base de datos. DAG, Shadow Redundancy y Safety Net se complementan; ninguna de las tres funciones sustituye una copia de seguridad frente a eliminaciones accidentales o daños prolongados no detectados ([Alta disponibilidad del transporte](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability)).

## Acceso de clientes y Autodiscover

La base de datos puede estar en buen estado y, aun así, un usuario no poder abrir Outlook. Los servicios de acceso de clientes aceptan conexiones HTTPS y las reenvían al backend del servidor que tiene la base de datos activa. Por tanto, un equilibrador de carga necesita más que un puerto TCP abierto: el nombre, el certificado, el punto de conexión del protocolo y el estado del backend deben coincidir ([Arquitectura del protocolo de acceso de clientes](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)).

Autodiscover proporciona al cliente la configuración adecuada. Los clientes internos del dominio pueden utilizar Service Connection Points en Active Directory; los clientes externos y otros clientes siguen procedimientos de DNS y HTTPS. Los errores suelen deberse a SCP obsoletos, respuestas DNS contradictorias, nombres de certificado incorrectos o un frontend que reenvía al backend equivocado ([Servicio Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

MAPI over HTTP es el transporte habitual de Outlook. Outlook en la Web, EWS y ActiveSync también utilizan HTTPS, pero tienen sus propios directorios virtuales, autenticación y propiedades de aplicación. Por tanto, una prueba satisfactoria de OWA no demuestra automáticamente una sesión MAPI/HTTP sana ([MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)).

## Active Directory y destinatarios

Después del transporte y el acceso de clientes, el directorio sigue siendo la base común. Exchange almacena la configuración de la organización y los servidores, así como los atributos de destinatarios, en Active Directory. Los cmdlets no escriben estos datos en una base de datos privada de Exchange, sino en AD mediante la lógica de Exchange ([Active Directory en Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)).

Por ello, un problema de destinatario se investiga a partir de tres preguntas: ¿Existe el objeto correcto? ¿Son correctos el tipo, la dirección principal, las direcciones proxy y los atributos de destino? ¿Ha llegado el cambio al controlador de dominio que utiliza el servicio Exchange afectado? Solo entonces merece la pena buscar en el transporte.

Para los expertos se añaden catálogos globales, sitios de AD, Recipient Update, políticas de libreta de direcciones y atributos híbridos. Los cambios directos con herramientas genéricas de AD omiten la validación de Exchange y pueden generar configuraciones presentes desde el punto de vista sintáctico, pero incoherentes desde el punto de vista funcional.

## Seguridad y control administrativo

Exchange publica servicios SMTP y HTTPS y procesa datos de directorio y buzones con altos privilegios. La base consiste en actualizaciones de seguridad oportunas, puntos de conexión mínimamente expuestos, certificados adecuados, cuentas administrativas protegidas y cambios trazables ([Actualizaciones de seguridad de Exchange Server](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates), [Certificados TLS en Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/client-access/certificates)).

RBAC separa las tareas mediante roles, grupos de roles y ámbitos. Los permisos de buzón como Full Access o Send As permanecen separados de ello. Administrator Audit Logging registra los cambios de cmdlets, pero no sustituye los registros del sistema operativo, Active Directory y seguridad ([Permisos en Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions), [Registro de auditoría de administradores](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/admin-audit-logging/admin-audit-logging)).

Para los expertos, la propia interfaz de administración forma parte del modelo de protección. EAC, Exchange Management Shell, PowerShell remoto, WinRM, RDP y el acceso al hipervisor tienen diferentes permisos y protocolos. Un administrador de servidor comprometido puede realizar acciones fuera de Exchange RBAC; por ello, la segmentación por niveles y las cuentas privilegiadas independientes siguen siendo importantes.

## Operación: del síntoma al servidor concreto

Managed Availability ejecuta sondas, monitores y respondedores. Los conjuntos de estado agrupan estos resultados por función y pueden activar acciones de recuperación automáticas. Son un buen punto de partida, pero no una comprobación integral de extremo a extremo ([Managed Availability](https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability)).

Para el flujo de correo, el diagnóstico local comienza con [`Get-Queue`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue) y [`Get-MessageTrackingLog`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog). El número de colas, el siguiente salto, la hora de reintento y `LastError` deben considerarse conjuntamente. Para las bases de datos se utilizan [`Get-MailboxDatabaseCopyStatus`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-mailboxdatabasecopystatus) y [`Test-ReplicationHealth`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/test-replicationhealth). [`Get-ServerHealth`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-serverhealth) muestra los conjuntos de estado y los monitores.

Estos cmdlets se ejecutan en Exchange Management Shell en versiones compatibles de Windows Server. En cambio, las pruebas de red y DNS pueden realizarse desde ambas plataformas de administración. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) comprueba un punto de conexión TCP en Windows; [`nc`](https://man.openbsd.org/nc) realiza la misma prueba de puerto en Unix. [`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) y [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) comprueban DNS. Para SMTP con STARTTLS resulta adecuado [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/), y para un diálogo SMTP controlado, [`swaks`](https://jetmore.org/john/code/swaks/).

El orden de diagnóstico es: resolver el nombre público o interno, comprobar la conexión con el frontend correcto, confirmar la aceptación en el registro de protocolo, seguir los eventos de seguimiento, comprobar la cola y el siguiente salto, y solo en caso de entrega local investigar el Store y la base de datos.

## Copia de seguridad y recuperación

La alta disponibilidad mantiene el servicio disponible ante fallos individuales; la recuperación restablece un estado anterior deseado o perdido. Exchange documenta Server Recovery, la restauración de bases de datos y Recovery Database como procedimientos distintos ([Copia de seguridad, restauración y recuperación ante desastres](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)).

Un inventario recuperable incluye como mínimo Active Directory, la organización y configuración de servidores Exchange, certificados y claves privadas, bases de datos de buzones con registros, configuración de conectores y reglas, así como parámetros documentados de instalación y recuperación. Recovery Database permite montar una base de datos restaurada de forma aislada y transferir contenidos a buzones activos ([Restaurar datos mediante una base de datos de recuperación](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/restore-data-using-recovery-dbs)).

Los expertos no solo comprueban si un trabajo de copia de seguridad se ha completado correctamente. Miden cuánto tiempo se necesita realmente para restaurar Active Directory, un servidor averiado, una base de datos y contenidos individuales de buzones. Durante ello se verifica qué secuencias de registros se necesitan, qué dependencias de DNS y certificados existen y si, tras la restauración, las rutas de clientes y SMTP vuelven a funcionar.

## Evolución técnica y límites

Exchange 4.0 apareció en 1996. Las primeras versiones utilizaban un directorio propio, MAPI y ESE; SMTP y Active Directory se convirtieron en componentes centrales de la plataforma con Exchange 2000. Exchange 2007 introdujo los roles de servidor y Exchange Management Shell. Exchange 2010 sustituyó los modelos de clúster anteriores por Database Availability Group ([Exchange Team: una breve historia del tiempo](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388), [Exchange Server 2007: rediseño del transporte](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)).

Las versiones posteriores volvieron a consolidar las funciones de Client Access y Mailbox en un componente común de servidor. Exchange Server Subscription Edition continuó la línea de productos locales en 2025 dentro del ciclo de vida moderno. Las versiones de compilación, las rutas de actualización compatibles y las actualizaciones de seguridad se comprueban antes de cada cambio en la documentación actual de Microsoft ([Notas de la versión de Exchange Server SE](https://learn.microsoft.com/en-us/exchange/release-notes), [Números de compilación y fechas de lanzamiento de Exchange Server](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates)).

Exchange On-Premises es adecuado cuando la organización necesita controlar la operación de las bases de datos, las rutas de red y la integración local, y puede asumir la operación necesaria 24/7. La contrapartida son dependencias complejas, mantenimiento continuo de seguridad y responsabilidad de recuperación. Un único servidor puede parecer sencillo; un servicio Exchange robusto es siempre también un proyecto de Active Directory, red, certificados, almacenamiento y operaciones.

## Fuentes

- [Microsoft Learn – Documentación de Exchange Server](https://learn.microsoft.com/en-us/exchange/exchange-server)
- [Microsoft Learn – Arquitectura de Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/architecture)
- [Microsoft Learn – Requisitos del sistema de Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/system-requirements)
- [Microsoft Learn – Active Directory en Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)
- [Microsoft Learn – Servidores de Edge Transport](https://learn.microsoft.com/en-us/exchange/architecture/edge-transport-servers/edge-transport-servers)
- [Microsoft Learn – Flujo de correo y canalización de transporte](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-flow)
- [Microsoft Learn – Seguimiento de mensajes](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)
- [Microsoft Learn – Dominios aceptados en Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Conectores en servidores Exchange](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Enrutamiento de correo en Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/mail-routing/mail-routing)
- [Microsoft Learn – Registros de transacciones y archivos de punto de control](https://learn.microsoft.com/en-us/exchange/client-developer/backup-restore/transaction-logs-and-checkpoint-files-for-backup-and-restore-in-exchange)
- [Microsoft Learn – Colas y la base de datos de colas](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)
- [Microsoft Learn – Grupos de disponibilidad de bases de datos](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-availability-groups)
- [Microsoft Learn – Supervisar grupos de disponibilidad de bases de datos](https://learn.microsoft.com/en-us/exchange/high-availability/manage-ha/monitor-dags)
- [Microsoft Learn – Copias de bases de datos de buzones](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/database-copies)
- [Microsoft Learn – Active Manager](https://learn.microsoft.com/en-us/exchange/high-availability/database-availability-groups/active-manager)
- [Microsoft Learn – Alta disponibilidad del transporte](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-high-availability/transport-high-availability)
- [Microsoft Learn – Arquitectura del protocolo de acceso de clientes](https://learn.microsoft.com/en-us/exchange/architecture/client-access/client-access)
- [Microsoft Learn – Servicio Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Microsoft Learn – MAPI over HTTP](https://learn.microsoft.com/en-us/exchange/clients/mapi-over-http/mapi-over-http)
- [Microsoft Learn – Interfaces de administración de Exchange](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/admin-interface)
- [Microsoft Learn – Certificados TLS en Exchange Server](https://learn.microsoft.com/en-us/exchange/architecture/client-access/certificates)
- [Microsoft Learn – Permisos en Exchange Server](https://learn.microsoft.com/en-us/exchange/permissions/permissions)
- [Microsoft Learn – Registro de auditoría de administradores](https://learn.microsoft.com/en-us/exchange/policy-and-compliance/admin-audit-logging/admin-audit-logging)
- [Microsoft Learn – Managed Availability](https://learn.microsoft.com/en-us/exchange/high-availability/managed-availability/managed-availability)
- [Microsoft Learn – Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue)
- [Microsoft Learn – Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog)
- [Microsoft Learn – Get-MailboxDatabaseCopyStatus](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-mailboxdatabasecopystatus)
- [Microsoft Learn – Test-ReplicationHealth](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/test-replicationhealth)
- [Microsoft Learn – Get-ServerHealth](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-serverhealth)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc(1)](https://man.openbsd.org/nc)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – manual de dig](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Swaks – herramienta de prueba SMTP](https://jetmore.org/john/code/swaks/)
- [Microsoft Learn – Copia de seguridad, restauración y recuperación ante desastres](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/disaster-recovery)
- [Microsoft Learn – Restaurar datos mediante una base de datos de recuperación](https://learn.microsoft.com/en-us/exchange/high-availability/disaster-recovery/restore-data-using-recovery-dbs)
- [Exchange Team – Una breve historia del tiempo](https://techcommunity.microsoft.com/blog/exchange/a-brief-history-of-time---exchange-server-way/589388)
- [Exchange Team – Exchange Server 2007: rediseño del transporte](https://techcommunity.microsoft.com/blog/exchange/motivations-for-the-exchange-server-2007-transport-redesign/600465)
- [Microsoft Learn – Notas de la versión de Exchange Server SE](https://learn.microsoft.com/en-us/exchange/release-notes)
- [Microsoft Learn – Números de compilación y fechas de lanzamiento de Exchange Server](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates)
