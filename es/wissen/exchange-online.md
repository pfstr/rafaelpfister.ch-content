---
title: "Exchange Online: arquitectura, flujo de correo y operaciones"
blatt: "exchange-online"
description: "Exchange Online para administradores de mensajería: modelo de tenant y destinatarios, transporte de EOP, conectores, acceso de clientes, PowerShell y Graph, Message Trace, retención, seguridad y recuperación."
fakten:
  - label: Función del producto
    wert: Servicio en la nube de correo electrónico, calendario y directorio
    href: https://learn.microsoft.com/en-us/exchange/exchange-online
  - label: Plataforma
    wert: Microsoft 365
    href: https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-service-description
  - label: Recepción de correo
    wert: Exchange Online Protection y SMTP
    href: https://learn.microsoft.com/en-us/defender-office-365/eop-about
  - label: Destinatarios
    wert: Buzones, grupos, contactos, usuarios de correo y recursos
    href: https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online
  - label: Dominios
    wert: Authoritative o Internal Relay
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains
  - label: Enrutamiento
    wert: Conectores de entrada y salida, reglas y MX
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail
  - label: Acceso de clientes
    wert: Outlook, Outlook en la web, ActiveSync y casos especiales de IMAP/POP
    href: https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/clients-and-mobile-in-exchange-online
  - label: Identidad
    wert: Microsoft Entra ID y autenticación moderna
    href: https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/enable-or-disable-modern-auth-in-exchange-online
  - label: Administración
    wert: Centro de administración de Exchange y Exchange Online PowerShell
    href: https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell
  - label: API
    wert: Microsoft Graph para funciones de correo, calendario y administración
    href: https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview
  - label: Diagnóstico
    wert: Message Trace, informes y Service Health
    href: https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message
  - label: Retención
    wert: Recoverable Items, Retention y Holds
    href: https://learn.microsoft.com/en-us/purview/retention-policies-exchange
werbung:
  - tools
  - newsletter
ctaThemen:
  - microsoft-365-exchange
  - smtp-mailflow
translationSourceHash: 5965bf4a9447505ffbe8b9a5d00c6f1abf629dfbde070d9c7700d9fdc9e4b603
translationModel: gpt-5.6-terra
translatedAt: 2026-10-05T11:29:35.054Z
translationReview: automatic
---

# Exchange Online: arquitectura, flujo de correo y operaciones

**Exchange Online** es el servicio Exchange operado por Microsoft en Microsoft 365. Proporciona buzones, calendarios, contactos, grupos, transporte SMTP y funciones de administración. El administrador del tenant decide sobre destinatarios, dominios, conectores, reglas, permisos y retención. Microsoft, en cambio, opera los servidores de buzones, las copias de bases de datos, las colas internas, los parches y los procesos de conmutación por error ([Exchange Online service description](https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-service-description), [Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)).

Por tanto, Exchange Online se parece funcionalmente a un sistema Exchange propio, pero no desde el punto de vista operativo. Un administrador local puede examinar un archivo de cola o activar una copia de base de datos. En Exchange Online, en su lugar, ve los eventos, estados y objetos de configuración proporcionados por el servicio. Por ello, la capacidad más importante es asignar una queja de usuario a una ruta clara: identidad, acceso de cliente, objeto destinatario, transporte, filtrado, entrega o retención.

## Del tenant al buzón

El tenant constituye el marco organizativo. En él, Exchange Online administra destinatarios habilitados para correo: buzones de usuario y compartidos, buzones de sala y equipo, listas de distribución, grupos de Microsoft 365, contactos y usuarios de correo. El tipo de destinatario determina si se almacenan datos, cómo se realiza la entrega y qué permisos están disponibles ([Recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online)).

Una cuenta de usuario en Microsoft Entra ID y un buzón de Exchange están vinculados, pero no son el mismo objeto. La asignación de licencias puede desencadenar el aprovisionamiento de un buzón. Exchange añade entonces atributos y servicios relacionados con el correo. Si un administrador elimina una licencia o borra una cuenta, se aplican distintos períodos de retención y eliminación. Por ello, para las operaciones y el offboarding, el ciclo de vida de la identidad, el ciclo de vida del buzón y la retención de cumplimiento deben planificarse conjuntamente ([Delete or restore user mailboxes](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/delete-or-restore-mailboxes), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

Para los expertos, el origen de los atributos adquiere importancia. En un tenant exclusivamente en la nube, las propiedades de Exchange se administran en línea. En las identidades sincronizadas, el entorno local puede seguir siendo la fuente autoritativa de determinados atributos de destinatarios. En ese caso, un valor aparece en Exchange Online, pero debe modificarse localmente y sincronizarse de nuevo. Este modelo corresponde al artículo [Exchange Hybrid](/kb/exchange-hybrid), ya que no existe sin sincronización de directorios.

## Cómo llega un mensaje entrante al buzón

Una vez entendido el destinatario, se puede seguir la ruta del correo. El MX público de un dominio normalmente apunta a Exchange Online Protection, EOP. EOP acepta la conexión SMTP, evalúa el remitente y el mensaje, aplica reglas de protección y transporte y entrega un mensaje permitido a Exchange Online. Para los destinatarios locales, a continuación se realiza la entrega en el buzón ([Exchange Online Protection overview](https://learn.microsoft.com/en-us/defender-office-365/eop-about), [Mail flow in EOP](https://learn.microsoft.com/en-us/defender-office-365/eop-mail-flow)).

El **Accepted Domain** determina cómo trata Exchange Online el dominio del destinatario. Con `Authoritative` el servicio espera todos los destinatarios válidos en la propia organización y rechaza las direcciones desconocidas. `Internal Relay` permite reenviar destinatarios desconocidos a otro sistema. Esta configuración solo tiene sentido si el siguiente salto y la resolución de destinatarios están planificados de forma fiable; de lo contrario, se producen errores de entrega o bucles ([Manage accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)).

Un mensaje interno no permanece automáticamente «en el mismo servidor». Exchange Online resuelve remitente y destinatario, comprueba reglas y directivas de protección y registra eventos de transporte. Para el administrador, esta cadena de eventos es decisiva: `Delivered` significa que el servicio ha entregado al destino; `Filtered`, `Failed`, `Pending` o `Expanded` describen otros pasos. Message Trace hace visibles estos pasos, pero no sustituye la comprobación del buzón de destino o de una regla posterior ([Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message), [Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)).

## Mensajes salientes y conectores

En los mensajes salientes, primero se decide si Exchange Online envía directamente al sistema de destino o utiliza un Outbound Connector configurado. Un conector puede dirigir mensajes a la propia infraestructura, a un socio o a una puerta de enlace de correo. La selección se basa, entre otros factores, en el dominio del destinatario, las condiciones del conector y las reglas de transporte ([Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)).

Los Inbound Connectors describen, a la inversa, en qué condiciones Exchange Online confía en un sistema remitente. Los criterios habituales son la IP de origen o un certificado TLS. Estos datos son relevantes para la seguridad: un rango IP demasiado amplio o un certificado comprobado de forma imprecisa puede hacer que tráfico externo parezca tráfico de socios internos.

Si una puerta de enlace de correo externa está delante de EOP, Microsoft ve primero la IP de la puerta de enlace. **Enhanced Filtering for Connectors** puede incluir información sobre el salto original en la evaluación del filtrado. La función no es un «interruptor de filtro antispam» general, sino que debe ajustarse a la ruta real, los conectores y las IP omitidas ([Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)).

La pregunta experta aquí es: ¿qué extremo aceptó realmente un mensaje, qué identidad se comprobó para el conector y en qué salto tuvo lugar el último filtrado de contenido? Estas tres respuestas deben figurar en cada diagrama de flujo de correo.

## Estructura técnica desde la perspectiva del administrador

Exchange Online no publica una lista de servidores que un administrador de tenant gestione como una granja local. No obstante, el servicio tiene componentes técnicos claramente reconocibles. Se hacen visibles mediante protocolos e interfaces de administración.

| Componente | Función | Lo que ve el administrador del tenant |
|---|---|---|
| Exchange Online Protection | Recepción SMTP, antimalware, antispam y procesamiento de transporte | Cuarentena, directivas, informes y Message Trace |
| Transporte de Exchange | Resolución de destinatarios, reglas, enrutamiento y entrega | Conectores, Accepted Domains, reglas y eventos |
| Servicio de buzones | Almacenamiento de correo electrónico, calendario, contactos y carpetas | Objetos de buzón, cuotas, permisos y acceso de clientes |
| Microsoft Entra ID | Identidades de usuarios, grupos, aplicaciones e inicio de sesión | Cuentas, roles, Conditional Access y registros de aplicaciones |
| Exchange Online PowerShell | Administración específica de Exchange | Cmdlets, RBAC y cambios auditables |
| Microsoft Graph | API REST para aplicaciones y automatización | Permisos OAuth, recursos y limitación de solicitudes |

La pila tecnológica en el borde consiste, por tanto, principalmente en SMTP y TLS para el transporte de correo, así como HTTPS, OAuth, PowerShell y REST para el acceso de clientes y administración. Los detalles internos de implementación solo son relevantes para el cliente en la medida en que Microsoft los documenta como comportamiento del servicio, límite o interfaz de diagnóstico ([About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2), [Microsoft Graph mail API](https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1056" src="/images/kb-interaktiv-exchange-online.svg?v=20260813" title="Interaktive Infografik: Exchange-Online-Pfad von DNS und EOP über Transport und Postfach bis Entra, PowerShell, Graph und Betriebsnachweis" loading="lazy">
  <a href="/images/kb-interaktiv-exchange-online.svg?v=20260813">Abrir directamente el gráfico interactivo de Exchange Online</a>.
</iframe>

## Acceso de clientes y autenticación moderna

El transporte de correo termina en el buzón; posteriormente, los usuarios acceden a él mediante protocolos de cliente. Outlook, Outlook en la web, los clientes móviles y las aplicaciones usan puntos de conexión basados en HTTPS. Autodiscover ayuda a los clientes a encontrar el servicio adecuado. El inicio de sesión se realiza mediante Microsoft Entra ID, mientras Exchange comprueba el permiso en el buzón ([Clients and mobile in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/clients-and-mobile-in-exchange-online), [Modern authentication in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/enable-or-disable-modern-auth-in-exchange-online)).

Esto separa dos errores que se confunden con frecuencia. Si el inicio de sesión falla en Entra, el cliente a menudo ni siquiera llega a Exchange. Si el token es válido, Exchange puede aun así denegar el acceso por falta de rol, permiso de buzón, directiva de cliente o un buzón de destino incorrecto. Por ello, el registro de inicio de sesión y el diagnóstico de Exchange deben considerarse conjuntamente en el tiempo.

Las aplicaciones acceden preferiblemente mediante Microsoft Graph o interfaces de Exchange compatibles. Un permiso de aplicación de Graph puede tener un alcance amplio; Exchange RBAC for Applications puede definir de forma más restrictiva el ámbito de buzones accesibles. Por tanto, un token OAuth válido es solo el primer paso. Después, el servicio de recursos comprueba qué acción está permitida en qué buzón ([Role Based Access Control for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)).

## Permisos y seguimiento de cambios

Exchange Online cuenta con sus propios roles administrativos. Los roles de Entra pueden permitir el acceso inicial a la administración de Exchange, pero los cmdlets reales de Exchange y su ámbito se determinan mediante Exchange-RBAC ([Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)).

Además, existen permisos de buzón como Full Access, Send As y Send on Behalf. Controlan acciones diferentes y no deben inventariarse como un único «derecho de delegación». Para las aplicaciones se añaden los roles de OAuth y de aplicación de Exchange ([Manage permissions for recipients](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)).

Para los expertos, el origen de los cambios es tan importante como el estado final. Los registros de auditoría, los registros de inicio de sesión de Entra y las exportaciones de configuración responden quién cambió una regla, un conector o un permiso. Una exportación nocturna de objetos centrales de flujo de correo facilita las comparaciones, pero no sustituye una fuente de auditoría protegida.

## Diagnóstico: primero DNS, luego eventos de transporte

Un análisis de flujo de correo comienza fuera del tenant. El MX indica qué sistema acepta correo de Internet. A continuación, se comprueba con Message Trace si Exchange Online ha visto el mensaje concreto y cómo lo ha procesado.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für MX- und Autodiscover-Abfrage">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.com
Resolve-DnsName autodiscover.example.com
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig MX example.com
dig autodiscover.example.com
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) y [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) muestran la publicación y la resolución. Aún no indican si EOP aceptó el mensaje o si un buzón lo recibió.

Para el siguiente paso, se elige un período acotado con remitente y destinatario. El mismo Exchange Online PowerShell se ejecuta en Windows y con `pwsh` en sistemas Unix compatibles.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Exchange Online PowerShell">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Connect-ExchangeOnline
Get-MessageTraceV2 -SenderAddress sender@example.net `
  -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date)
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```powershell
pwsh
Connect-ExchangeOnline
Get-MessageTraceV2 -SenderAddress sender@example.net `
  -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date)
```

  </div>
</div>

[`Connect-ExchangeOnline`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/connect-exchangeonline) establece la sesión de administración autenticada. [`Get-MessageTraceV2`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2) busca eventos de transporte; [`Get-Date`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-date) limita la ventana temporal. Para tendencias se añaden informes y, para incidencias de Microsoft, Service Health. Una única señal verde no responde a las tres preguntas ([Exchange Online monitoring](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-exchange-monitoring?view=o365-worldwide)).

## Retención, eliminación y recuperación

Microsoft protege el servicio en funcionamiento con varias copias de bases de datos, Shadow Redundancy y Safety Net. Estos mecanismos sirven a la disponibilidad y la integridad de los datos del servicio. No son la interfaz de usuario para recuperar un mensaje eliminado accidentalmente ([Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)).

Para casos de usuarios y cumplimiento se aplican otras funciones: Deleted Item Retention, Recoverable Items, Single Item Recovery, Retention Policies y Holds. Sus efectos se solapan, pero tienen finalidades distintas. Una regla de retención puede proteger el contenido frente a la eliminación definitiva; no proporciona automáticamente una copia de seguridad independiente del tenant con un momento de recuperación libremente elegible ([Recoverable Items folder](https://learn.microsoft.com/en-us/exchange/security-and-compliance/recoverable-items-folder/recoverable-items-folder), [Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)).

Por ello, un concepto de recuperación sólido documenta qué eventos cubre la resiliencia del servicio de Microsoft, qué contenidos pueden recuperarse mediante la retención de Exchange o Purview y para qué requisitos se necesita una copia independiente. Las pruebas de restauración deben utilizar casos concretos: mensaje individual, carpeta, buzón tras eliminar un usuario, elemento retenido por motivos legales e incidencia en todo el tenant.

## Seguridad y límites habituales

Exchange Online conecta varios ámbitos de seguridad: correo de Internet, EOP, configuración del tenant, inicio de sesión de Entra, permisos de buzón y aplicaciones. La eficacia de la protección depende de que la ruta real de los mensajes y del inicio de sesión coincida con la configuración.

Para el flujo de correo, esto significa que MX, identidad del conector, Enhanced Filtering, SPF/DKIM/DMARC y reglas de transporte deben comprobarse como una cadena. Para el acceso de clientes, la autenticación moderna, Conditional Access, Exchange-RBAC y los permisos de buzón son controles separados. Para las aplicaciones se añaden el consentimiento OAuth y el ámbito de buzones permitido.

La pregunta administrativa más profunda es siempre la misma: ¿qué sistema tomó la decisión, qué datos de entrada vio y dónde está registrado el resultado? Sin estos tres datos, incluso una directiva formalmente correcta es difícil de comprobar.

## Evolución técnica y compromisos deliberados

Exchange Online evolucionó a partir de las ofertas Exchange alojadas de Microsoft y adoptó muchos conceptos del producto de servidor: destinatarios, bases de datos de buzones, transporte, DAG, Shadow Redundancy y Safety Net. El servicio automatiza la operación de esta infraestructura y proporciona a los administradores de tenant un nivel superior de administración ([Exchange Team: 20 years ago](https://techcommunity.microsoft.com/blog/exchange/20-years-ago-in-a-galaxy-far-away8230/604456), [Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)).

La ventaja reside en la operación externalizada de la plataforma, la integración global de servicios y las interfaces de administración estandarizadas. El precio es un menor acceso a servidores individuales, colas y copias de bases de datos, así como una mayor dependencia de las funciones publicadas de diagnóstico, exportación y recuperación. Por ello, la tarea de los expertos no consiste en adivinar la topología interna invisible, sino en utilizar plenamente los controles del tenant y las señales del servicio documentados.

## Fuentes

- [Microsoft Learn – Exchange Online](https://learn.microsoft.com/en-us/exchange/exchange-online)
- [Microsoft – Exchange Online service description](https://learn.microsoft.com/en-us/office365/servicedescriptions/exchange-online-service-description/exchange-online-service-description)
- [Microsoft Defender – Exchange Online Protection overview](https://learn.microsoft.com/en-us/defender-office-365/eop-about)
- [Microsoft Defender – Mail flow in EOP](https://learn.microsoft.com/en-us/defender-office-365/eop-mail-flow)
- [Microsoft Learn – Recipients in Exchange Online](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/recipients-in-exchange-online)
- [Microsoft Learn – Delete or restore user mailboxes](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/delete-or-restore-mailboxes)
- [Microsoft Learn – Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)
- [Microsoft Learn – Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)
- [Microsoft Learn – Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)
- [Microsoft Learn – Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)
- [Microsoft Learn – Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)
- [Microsoft Learn – Clients and mobile in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/clients-and-mobile-in-exchange-online)
- [Microsoft Learn – Modern authentication in Exchange Online](https://learn.microsoft.com/en-us/exchange/clients-and-mobile-in-exchange-online/enable-or-disable-modern-auth-in-exchange-online)
- [Microsoft Learn – Exchange Online PowerShell](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell)
- [Microsoft Learn – About the Exchange Online PowerShell module](https://learn.microsoft.com/en-us/powershell/exchange/exchange-online-powershell-v2)
- [Microsoft Graph – Mail API overview](https://learn.microsoft.com/en-us/graph/api/resources/mail-api-overview)
- [Microsoft Learn – RBAC for Applications](https://learn.microsoft.com/en-us/exchange/permissions-exo/application-rbac)
- [Microsoft Learn – Permissions in Exchange Online](https://learn.microsoft.com/en-us/exchange/permissions-exo/permissions-exo)
- [Microsoft Learn – Manage permissions for recipients](https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/manage-permissions-for-recipients)
- [Microsoft Service Assurance – Exchange Online data resiliency](https://learn.microsoft.com/en-us/compliance/assurance/assurance-exchange-data-resiliency)
- [Microsoft Learn – Recoverable Items folder](https://learn.microsoft.com/en-us/exchange/security-and-compliance/recoverable-items-folder/recoverable-items-folder)
- [Microsoft Purview – Retention policies for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange)
- [Microsoft Learn – Exchange Online monitoring](https://learn.microsoft.com/en-us/microsoft-365/enterprise/microsoft-365-exchange-monitoring?view=o365-worldwide)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- [Microsoft Learn – Connect-ExchangeOnline](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/connect-exchangeonline)
- [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2)
- [Microsoft Learn – Get-Date](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-date)
- [Exchange Team – 20 years ago](https://techcommunity.microsoft.com/blog/exchange/20-years-ago-in-a-galaxy-far-away8230/604456)
