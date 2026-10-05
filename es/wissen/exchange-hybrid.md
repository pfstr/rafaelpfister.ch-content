---
title: "Exchange Hybrid: identidad, coexistencia y operación"
blatt: "exchange-hybrid"
description: "Exchange Hybrid explicado de forma clara: requisitos, sincronización de directorios, autoridad de destinatarios, Hybrid Configuration Wizard, OAuth, relaciones de organización, Autodiscover, movimientos de buzones y operación."
fakten:
  - label: Propósito
    wert: Coexistencia de Exchange Server y Exchange Online
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Espacio de nombres compartido
    wert: Los buzones de ambos lados pueden usar los mismos dominios SMTP
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Sincronización de directorios
    wert: Microsoft Entra Connect Sync o Cloud Sync según el modelo compatible
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites
  - label: Herramienta de configuración
    wert: Hybrid Configuration Wizard
    href: https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard
  - label: Configuración local
    wert: Objeto HybridConfiguration en Active Directory
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Configuración en la nube
    wert: Conectores, relaciones de organización y confianza OAuth
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid
  - label: Modelo de destinatarios
    wert: Remote Mailbox local, buzón de Exchange Online en la nube
    href: https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox
  - label: Funciones de coexistencia
    wert: Libre/ocupado, MailTips, archivado, búsqueda y movimientos de buzones según la configuración
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Migración de buzones
    wert: Mailbox Replication Service y puntos de conexión de migración
    href: https://learn.microsoft.com/en-us/exchange/mailbox-migration/migrate-mailboxes-across-tenants
  - label: Transporte de correo
    wert: SMTP/TLS basado en certificados entre ambas organizaciones
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites
  - label: Administración de destinatarios
    wert: Exchange Management Tools o transferencia de SoA a la nube compatible
    href: https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools
  - label: Diagnóstico
    wert: Registro de HCW, estado de sincronización de Entra, configuración de OAuth, organización y transporte
    href: https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - microsoft-365-exchange
translationSourceHash: c35b1646509133dd8975f96d030b2990225af14469d14a61e4fd16a4e2737f08
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:20:02.336Z
translationReview: automatic
---

# Exchange Hybrid: identidad, coexistencia y operación

**Exchange Hybrid** conecta una organización de Exchange local con Exchange Online. Los usuarios pueden tener buzones en ambos lados y, aun así, utilizar los mismos dominios SMTP, una libreta de direcciones común y determinadas funciones entre organizaciones. Por ello, Hybrid es más que un par de conectores: conecta datos de directorio, destinatarios, autenticación, Autodiscover, funciones de calendario, migración y transporte de correo ([Implementaciones híbridas de Exchange](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Aclare primero tres preguntas: **¿Dónde está el buzón?** **¿Dónde se administra su objeto de destinatario?** Y **¿qué servicio ejecuta la operación solicitada?** Cuando estas tres respuestas están claras, los numerosos componentes híbridos se convierten en una cadena comprensible.

## Lo que Hybrid reúne para los usuarios

Sin Hybrid, la organización local de Exchange y Exchange Online son dos sistemas separados. Hybrid crea una experiencia de usuario común. Los buzones pueden utilizar el mismo dominio SMTP principal. La información de la libreta de direcciones se sincroniza. Las consultas de libre/ocupado y MailTips pueden funcionar entre organizaciones. Los buzones se pueden mover mediante Remote Moves compatibles ([Implementaciones híbridas de Exchange](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Sin embargo, estas funciones no comparten un único almacén de datos común. Un buzón local permanece en una base de datos ESE local; un buzón en la nube permanece en Exchange Online. Active Directory y Entra ID mantienen objetos de directorio respectivamente. Las relaciones de organización y OAuth permiten consultas seleccionadas a través de ese límite. Los conectores SMTP transportan mensajes. El visible «un solo Exchange» surge de conexiones coordinadas.

Para los administradores, de ello se deriva una regla importante: un flujo de correo correcto no demuestra que libre/ocupado funcione, y una consulta de libre/ocupado correcta no demuestra que sea posible un Remote Move. Cada función tiene su propio camino y sus propias evidencias.

## Los componentes en un orden lógico

Una implementación híbrida comienza con sus requisitos, no con el asistente. La organización local de Exchange debe estar en una versión compatible. Los nombres públicos, certificados, DNS, accesibilidad HTTPS y SMTP deben ser correctos. Un tenant de Microsoft 365 con Exchange Online y una sincronización de directorios compatible conectan posteriormente las identidades ([Requisitos previos para implementaciones híbridas](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

Sobre ello se basa el **Hybrid Configuration Wizard**, HCW. Lee la configuración deseada, escribe un objeto `HybridConfiguration` en el Active Directory local y configura los ajustes adecuados localmente y en Exchange Online. Estos pueden incluir relaciones de organización, OAuth, conectores intraorganizativos y conectores de transporte ([Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard), [Crear una implementación híbrida](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid)).

| Componente | Tarea básica | Qué se comprueba primero ante un fallo |
|---|---|---|
| Active Directory | atributos locales de usuarios y Exchange | Objeto, tipo de destinatario, direcciones proxy y hora de modificación |
| Sincronización de Entra | transfiere atributos compatibles de identidad y destinatarios | Errores de exportación, estado de sincronización y objeto en la nube |
| Exchange Online | buzón y configuración en la nube | Tipo de destinatario, licencia, estado del buzón y RBAC |
| Configuración de HCW | coordina las dos organizaciones de Exchange | Registro de HCW, parámetros seleccionados y objetos modificados posteriormente |
| Relación de organización y OAuth | funciones entre organizaciones | URI de destino, Autodiscover, certificados y flujo de tokens |
| Conectores SMTP | mensajes entre ambos lados | Nombre del certificado, host de origen/destino, TLS y Message Trace |

La tabla también muestra por qué «ejecutar HCW de nuevo» no es una reparación universal. El asistente puede volver a coordinar objetos híbridos documentados. No repara una zona DNS defectuosa, una ruta de firewall bloqueada ni un objeto de destinatario mantenido incorrectamente.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-exchange-hybrid.svg?v=20260813" title="Interaktive Infografik: Exchange-Hybrid-Verbindungen für Verzeichnissync, Empfänger, HCW, OAuth, Frei-Gebucht, Mailboxverschiebung und SMTP" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-hybrid.svg?v=20260813">Abrir directamente el gráfico interactivo de Exchange Hybrid</a>.
</iframe>

## Pila tecnológica: protocolos y herramientas de administración

Hybrid no es un proceso adicional de servidor Exchange, sino una conexión de sistemas existentes. Active Directory y Entra ID mantienen identidades y atributos de destinatarios. La sincronización de Entra transfiere valores compatibles. HTTPS transporta Autodiscover, libre/ocupado, llamadas de servicio protegidas por OAuth y movimientos de buzones. SMTP con TLS transporta mensajes. PowerShell, Exchange Admin Center y HCW administran los objetos implicados ([Requisitos previos para implementaciones híbridas](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites), [Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

Esta división también determina el orden ante incidencias. Un error de objeto se busca en el directorio y la sincronización, un problema de calendario en la ruta HTTPS/OAuth y un problema de correo en SMTP y los conectores. Así, el conjunto de herramientas permanece vinculado a la función afectada.

## Sincronización de directorios y autoridad de destinatarios

Una vez conectadas las plataformas, el origen de los datos de destinatarios se convierte en la cuestión operativa más importante. En entornos híbridos clásicos, un usuario se crea en el Active Directory local. Las herramientas de Exchange escriben los atributos relacionados con el correo. Entra Connect sincroniza el objeto con la nube, donde Exchange Online proporciona el objeto en la nube correspondiente y, en su caso, un buzón ([Requisitos previos para implementaciones híbridas](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

Un **Remote Mailbox** es un objeto local habilitado para correo que hace referencia a un buzón de Exchange Online. Atributos como `remoteRoutingAddress`, `proxyAddresses` y el tipo de destinatario ayudan a la organización local a dirigir mensajes y administración hacia la nube. [`Enable-RemoteMailbox`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox) crea o habilita esta representación local; el buzón en la nube solo se crea mediante sincronización y asignación de licencia.

Por tanto, la pregunta habitual del administrador es: ¿dónde debo modificar este valor? La pregunta experta es: ¿qué sistema es autoritativo para **este atributo concreto** y qué ciclo de sincronización lo transfiere? Un portal en la nube puede mostrar un valor sincronizado sin permitir editarlo de forma permanente.

Microsoft admite escenarios en los que solo permanecen las Exchange Management Tools para los atributos locales de destinatarios. Para determinados entornos, también existe un procedimiento para transferir a la nube la administración de atributos de Exchange. Son modelos operativos distintos con requisitos; desactivar el último servidor por sí solo no transfiere la autoridad sobre los datos ([Administrar destinatarios con Exchange Management Tools](https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools), [Retirar después de la transferencia de Source of Authority](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/decommission-last-exchange-server)).

## Autodiscover y la ruta del cliente

Cuando los destinatarios son correctos, un cliente debe encontrar la ubicación del buzón. Autodiscover responde a esta pregunta. Los puntos de conexión locales de Exchange pueden redirigir a un cliente con un buzón en la nube a Exchange Online; los puntos de conexión en la nube proporcionan la configuración del buzón en línea ([Autodiscover en implementaciones híbridas de Exchange](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)).

Por ello, un problema de Autodiscover híbrido suele manifestarse como una ubicación incorrecta: el usuario puede iniciar sesión en principio, pero llega al punto de conexión local, recibe una redirección inesperada o recibe configuraciones para un buzón que ya no existe. DNS, SCP, directorios virtuales, certificados y atributos de destinatarios se comprueban en este orden.

Solo después de que el cliente haya llegado al servicio de buzón correcto tienen sentido las cuestiones de protocolo y permisos. Así, el diagnóstico sigue siendo comprensible: primero encontrar, después iniciar sesión y luego autorizar.

## Libre/ocupado y otras funciones entre organizaciones

Una libreta de direcciones común no basta para las consultas de calendario. Libre/ocupado requiere relaciones de organización, Autodiscover accesible y una configuración de confianza u OAuth funcional. Exchange consulta información en el otro lado en lugar de copiar completamente los datos de calendario en su propio sistema ([Uso compartido en implementaciones híbridas de Exchange](https://learn.microsoft.com/en-us/exchange/sharing/sharing)).

El mismo patrón básico se aplica a otras funciones híbridas: un componente local realiza una solicitud, la contraparte la autentica, autoriza la operación y devuelve un resultado limitado. Por ello, durante la resolución de problemas se registran el buzón de origen, el buzón de destino, la dirección y el punto de conexión. «Libre/ocupado no funciona» es demasiado impreciso sin estos datos.

Para los expertos, los tokens y los URI de destino se vuelven relevantes. HCW configura relaciones entre organizaciones, pero los cambios de certificados, las modificaciones manuales o los puntos de conexión obsoletos pueden afectar a la operación posterior. La configuración de ambos lados siempre se exporta conjuntamente.

## OAuth entre las organizaciones de Exchange

Una vez claro qué consultas entre organizaciones se realizan, se puede clasificar su autenticación. Exchange puede utilizar OAuth para que una organización acredite una llamada de servicio ante la otra. Esto afecta a funciones híbridas como disponibilidad entre organizaciones y determinadas operaciones de archivado, búsqueda o migración; el uso exacto depende de la versión y la configuración ([Configurar la autenticación OAuth](https://learn.microsoft.com/en-us/exchange/configure-oauth-authentication-between-exchange-and-exchange-online-organizations-exchange-2013-help)).

El flujo de tokens no sustituye a SMTP-TLS. OAuth protege llamadas de aplicaciones, mientras que el transporte de correo híbrido utiliza sus propios conectores y comprobaciones de certificados. Esta separación evita el salto que confunde en muchas explicaciones: primero se determina la función, luego su protocolo y solo después la autenticación.

Para los expertos, AuthConfig, AuthServer, PartnerApplication, Intra-Organization Connector y Organization Relationship forman parte de una visión de comprobación conjunta. Un objeto individual puede estar presente sintácticamente mientras el certificado, el realm o el URI de destino ya no coinciden con la contraparte.

## Hybrid Modern Authentication es un tema independiente del cliente

**Hybrid Modern Authentication**, HMA, se aborda solo ahora porque no explica el transporte de correo ni la sincronización de destinatarios. HMA permite que los recursos locales compatibles de Exchange y Skype for Business utilicen Microsoft Entra ID para la autenticación moderna de clientes. El cliente obtiene un token de Entra y lo utiliza ante el servicio local ([Descripción general de Hybrid modern authentication](https://learn.microsoft.com/en-us/microsoft-365/enterprise/hybrid-modern-auth-overview)).

Por tanto, HMA amplía el acceso de clientes con una dependencia de la nube. La accesibilidad de Entra, las URL publicadas, los Service Principal Names registrados y la configuración local de Exchange deben coincidir. Un flujo de correo híbrido funcional no dice nada sobre esta ruta de tokens.

Por ello, los expertos tratan HMA en un runbook independiente con versiones compatibles, exclusiones, grupos de despliegue y plan de reversión. La función no se añade de forma incidental a una opción de enrutamiento.

## Movimientos de buzones

La coexistencia se establece con frecuencia para mover buzones gradualmente. Un Remote Move copia datos del buzón mediante Mailbox Replication Service, mantiene los cambios sincronizados y cambia el buzón de forma controlada al lado de destino. Los atributos de destinatario y el enrutamiento se trasladan o ajustan en el proceso ([Mover buzones entre entornos locales y Exchange Online](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes)).

Para administradores avanzados, el proceso consta de preparación, inicio, sincronización, finalización y comprobación posterior. Antes de finalizar, se comprueban el volumen de datos, elementos con errores, delegaciones, archivos, acceso de clientes y flujo de correo. Después de finalizar, Autodiscover, licencia, dirección de destino y el objeto local Remote Mailbox deben coincidir.

Los expertos planifican tamaños de lotes, rendimiento de red, limitación de MRS, límites de elementos defectuosos, objetos grandes y migración inversa. El valor de progreso técnico por sí solo no equivale a una aceptación; también se incluyen el acceso de usuarios, las delegaciones, los clientes móviles y las funciones entre organizaciones.

## El flujo de correo híbrido sigue siendo una ruta independiente

Hybrid necesita SMTP entre la organización local y Exchange Online. Esta ruta de correo utiliza conectores, TLS y certificados. Es lo suficientemente importante como para un artículo propio, porque el correo de Internet, Centralized Mail Transport, las puertas de enlace de correo y los dominios compartidos forman varias variantes ([Requisitos previos para implementaciones híbridas](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

El artículo [Flujo de correo híbrido](/kb/hybrid-mailfluss) comienza con un mensaje concreto y sigue cada salto. Solo allí se comparan Centralized Mail Transport, IP de salida, ubicación del filtrado y colas adicionales. Este artículo se concentra en identidad y coexistencia.

## Seguridad y operación

Hybrid amplía los sistemas accesibles. Los puntos de conexión HTTPS y SMTP públicos, certificados, sincronización de Entra, cuentas con privilegios y objetos de confianza entre organizaciones deben inventariarse conjuntamente. HCW necesita permisos amplios en ambos lados; su uso y sus registros deben protegerse y archivarse de manera trazable ([Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

En el día a día, cada función híbrida debe tener un responsable y una prueba: sincronización de destinatarios, libre/ocupado en ambas direcciones, Remote Move, Autodiscover y SMTP en ambas direcciones. Una prueba sintética periódica detecta certificados vencidos o puntos de conexión modificados silenciosamente antes que un proyecto de migración.

Ante incidencias, ayuda una línea temporal común. Los eventos de sincronización de Entra, el registro de HCW, los registros de eventos de Exchange, las pruebas de OAuth, el seguimiento de mensajes y Message Trace no se recopilan indiscriminadamente, sino que se asignan a la función afectada. Esto acorta el diagnóstico y evita que una prueba correcta de otra función se interprete como evidencia.

## Copia de seguridad, reconstrucción y retirada

Los datos de los buzones se protegen en el lado donde se encuentran: bases de datos locales con recuperación local, buzones en la nube con funciones de Exchange Online y Purview. Además, la conexión debe poder restaurarse. Esto incluye el objeto local HybridConfiguration, certificados y claves privadas, configuración de conectores y organizaciones, reglas de sincronización de Entra y decisiones documentadas de HCW.

Una reconstrucción comienza con identidad y resolución de nombres, seguida de accesibilidad HTTPS y SMTP, configuración de HCW y pruebas funcionales. El asistente puede volver a crear la configuración, pero sin certificados, DNS y objetos de destinatarios adecuados no se obtiene un sistema global funcional.

Durante la retirada, primero se aclara qué funciones híbridas siguen utilizándose. Microsoft diferencia entre un servidor restante, herramientas de administración puras y la transferencia a la nube de la administración de atributos de Exchange. Solo después de esta decisión se eliminan de forma controlada conectores, relaciones de organización, puntos de conexión y servidores ([Administrar destinatarios con Exchange Management Tools](https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools), [Retirar después de la transferencia de Source of Authority](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/decommission-last-exchange-server)).

## Evolución técnica y límites

Hybrid surgió con Exchange Online como una forma de ampliar de manera controlada las organizaciones locales hacia el servicio en la nube. Las generaciones anteriores utilizaban en mayor medida Federation Trusts; las versiones más recientes de Exchange y los flujos de HCW usan OAuth e Intra-Organization Connectors para muchas funciones entre organizaciones ([Crear una implementación híbrida](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid)).

El modelo es potente porque permite la migración y la coexistencia permanente. Es exigente porque deben operarse ambas organizaciones de Exchange y su conexión. Por ello, quien ya no tenga buzones locales después de una migración debe decidir conscientemente qué función de administración o coexistencia sigue justificando Hybrid.

## Fuentes

- [Microsoft Learn – Implementaciones híbridas de Exchange](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Microsoft Learn – Active Directory en Exchange Server](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/active-directory/active-directory)
- [Microsoft Learn – Destinatarios en Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online)
- [Microsoft Learn – Requisitos previos para implementaciones híbridas](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)
- [Microsoft Learn – Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)
- [Microsoft Learn – Crear una implementación híbrida](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/deploy-hybrid)
- [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox)
- [Microsoft Learn – Administrar destinatarios con Exchange Management Tools](https://learn.microsoft.com/en-us/exchange/manage-hybrid-exchange-recipients-with-management-tools)
- [Microsoft Learn – Retirar después de la transferencia de Source of Authority](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/decommission-last-exchange-server)
- [Microsoft Learn – Servicio Autodiscover](https://learn.microsoft.com/en-us/exchange/architecture/client-access/autodiscover)
- [Microsoft Learn – Uso compartido en Exchange](https://learn.microsoft.com/en-us/exchange/sharing/sharing)
- [Microsoft Learn – Configurar la autenticación OAuth](https://learn.microsoft.com/en-us/exchange/configure-oauth-authentication-between-exchange-and-exchange-online-organizations-exchange-2013-help)
- [Microsoft Learn – Descripción general de Hybrid modern authentication](https://learn.microsoft.com/en-us/microsoft-365/enterprise/hybrid-modern-auth-overview)
- [Microsoft Learn – Mover buzones entre entornos locales y Exchange Online](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/move-mailboxes)
- [Microsoft Learn – Migración de buzones entre tenants](https://learn.microsoft.com/en-us/exchange/mailbox-migration/migrate-mailboxes-across-tenants)
