---
title: "Flujo de correo híbrido: enrutamiento entre Exchange Online y entorno local"
blatt: "hybrid-mailfluss"
description: "Flujo de correo híbrido paso a paso: entrega directa y centralizada, conectores, certificados TLS, dominios remotos, dominios SMTP compartidos, puertas de enlace de correo, Message Trace y Message Tracking."
fakten:
  - label: Función
    wert: Enrutamiento SMTP entre Exchange Online y la organización local de Exchange
    href: https://learn.microsoft.com/en-us/exchange/transport-routing
  - label: Configuración
    wert: Hybrid Configuration Wizard crea y mantiene la configuración de transporte
    href: https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard
  - label: Transporte
    wert: SMTP a través de TCP 25 con TLS
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites
  - label: Verificación de pares
    wert: Nombre del certificado y condiciones del conector
    href: https://learn.microsoft.com/en-us/exchange/hybrid-deployment/hybrid-mail-flow
  - label: Conectores de nube
    wert: Conectores entrantes y salientes
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail
  - label: Conectores locales
    wert: Conectores de recepción y envío
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors
  - label: Enrutamiento de destinatarios
    wert: Buzón remoto, Target Address y dominio de coexistencia
    href: https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox
  - label: Enrutamiento estándar
    wert: Las organizaciones en la nube y locales pueden enviar correo de Internet directamente
    href: https://learn.microsoft.com/en-us/exchange/transport-routing
  - label: Centralized Mail Transport
    wert: El correo de Internet de Exchange Online pasa por la organización local
    href: https://learn.microsoft.com/en-us/exchange/transport-routing
  - label: Puertas de enlace externas
    wert: Cadena adicional de conectores y filtrado antes o después de Microsoft 365
    href: https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud
  - label: Diagnóstico en la nube
    wert: Message Trace y validación de conectores
    href: https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message
  - label: Diagnóstico local
    wert: Message Tracking, colas y registros de protocolo
    href: https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking
werbung:
  - tools
  - newsletter
ctaThemen:
  - exchange-onprem-hybrid
  - smtp-mailflow
translationSourceHash: 442ec532c64e034403d3449ffe1b46d4b6de05a2337d3c3210da2d83cb5f618f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-07T11:03:54.736Z
translationReview: automatic
---

# Flujo de correo híbrido: enrutamiento entre Exchange Online y entorno local

El **flujo de correo híbrido** es la ruta SMTP entre una organización local de Exchange y Exchange Online. Permite que los buzones de ambos lados utilicen el mismo dominio SMTP y que los mensajes lleguen al lugar correcto. Para ello, Hybrid Configuration Wizard configura conectores y parámetros TLS ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing), [Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

El flujo de correo es solo una parte de Exchange Hybrid. La sincronización de directorios, libre/ocupado, OAuth y los movimientos de buzones utilizan otras rutas. Por ello, este artículo se limita deliberadamente a una única pregunta: **¿Qué saltos SMTP recorre un mensaje concreto y qué decisión se toma en cada salto?**

La **pila de protocolos** es clara: DNS nombra los destinos accesibles públicamente, SMTP a través de TCP 25 transporta el mensaje, TLS protege e identifica la conexión, y los conectores de Exchange determinan qué extremo se utiliza para cada dominio. Los objetos de destinatario proporcionan la dirección de enrutamiento; Message Trace y los registros de seguimiento locales muestran posteriormente qué hizo cada organización con el mensaje ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing), [Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)).

## El modelo básico: dos organizaciones de Exchange, un espacio de direcciones

Un entorno híbrido tiene al menos dos organizaciones de transporte. La organización local de Exchange conoce los buzones locales y los objetos de buzón remoto. Exchange Online conoce los buzones en la nube y las representaciones sincronizadas de destinatarios locales. Ambos lados pueden utilizar el mismo dominio principal, como `example.com` ([Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).

Para que un mensaje no termine en el lugar equivocado, cada lado necesita una indicación de la ubicación real del destinatario. Para un buzón en la nube, el objeto de buzón remoto local contiene una dirección de enrutamiento remoto en el dominio de coexistencia, normalmente `tenant.mail.onmicrosoft.com`. A la inversa, Exchange Online conoce los destinatarios locales sincronizados como objetos habilitados para correo ([Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox)).

El flujo normal es sencillo:

1. La primera organización de Exchange acepta el mensaje.
2. Resuelve el destinatario en su directorio.
3. El objeto de destinatario indica si el buzón está localmente o en el otro lado.
4. El conector híbrido adecuado envía mediante SMTP/TLS a la otra organización.
5. Allí se vuelve a resolver el destinatario y se entrega el mensaje.

Los expertos comprueban además si las reglas de transporte modifican la ruta, si se ha intercalado una puerta de enlace y qué prioridad de dominio o conector explica el siguiente salto elegido.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-hybrid-mailfluss.svg?v=20260813" title="Interaktive Infografik: direkter und zentraler Hybrid-Mailfluss zwischen Internet, Exchange Online, Exchange On-Premises und Mail-Gateway" loading="lazy">
  <a href="/images/kb-interaktiv-hybrid-mailfluss.svg?v=20260813">Abrir directamente el gráfico interactivo del flujo de correo híbrido</a>.
</iframe>

## La ruta estándar sin transporte centralizado de Internet

En el modelo descentralizado habitual, cada lado envía su propio correo de Internet. Un buzón local utiliza la organización de transporte local de Exchange para el correo saliente de Internet. Un buzón en la nube envía a través de Exchange Online Protection. Solo los mensajes entre buzones locales y en la nube pasan por los conectores híbridos ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing)).

El correo de Internet entrante sigue el MX publicado. Si el MX apunta a Exchange Online, EOP acepta primero el mensaje. Para un buzón en la nube, Exchange Online entrega localmente; para un destinatario local sincronizado, utiliza el conector saliente híbrido. Si, por el contrario, el MX apunta al entorno local o a una puerta de enlace anterior, la primera decisión de destinatario se toma allí.

Este modelo mantiene cortas las rutas de Internet, pero genera varias IP de salida y ubicaciones de filtrado posibles. SPF, DKIM, DMARC, las listas de permitidos y las reglas de socios deben tener en cuenta ambas rutas de salida. No es un error del modelo híbrido, sino una consecuencia de la entrega distribuida.

## Clasificar conscientemente Centralized Mail Transport

**Centralized Mail Transport**, CMT, modifica exactamente esta ruta saliente. Los mensajes desde buzones de Exchange Online hacia Internet se envían primero a la organización local de Exchange. Solo allí salen de la organización. Esto permite que el lado local siga utilizando reglas de transporte centralizadas, dispositivos o IP de salida fijas ([Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing)).

La ventaja es un control común de salida. El coste son saltos y dependencias adicionales. Si falla el transporte local o su conexión a Internet, esto también afecta ahora al correo saliente de la nube. Cambian la latencia, la ubicación de la cola, la IP de salida y el lugar del último filtrado.

Para administradores avanzados, la decisión no es por tanto «activar o desactivar CMT», sino: ¿Qué política concreta requiere el salto local, qué capacidad debe soportar y cómo se enruta en caso de fallo? Los expertos documentan además la protección contra bucles, los requisitos TLS, la prioridad de conectores y la prueba de que cada mensaje previsto realmente sigue la ruta centralizada.

Con ello queda concluido el tema de CMT. La autenticación de clientes de Outlook o Hybrid Modern Authentication no pertenece aquí, porque no selecciona ningún salto SMTP.

## Cómo reconocen los conectores al extremo remoto

Una vez elegida la ruta, cada lado debe poder confiar en el extremo remoto. Hybrid Configuration Wizard crea conectores de envío/recepción locales y conectores entrantes/salientes adecuados en Exchange Online. El transporte se realiza mediante SMTP en TCP 25 y utiliza TLS ([Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites), [Hybrid mail flow](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/hybrid-mail-flow)).

El conector de envío local determina el destino y los requisitos TLS para el dominio de coexistencia. El conector saliente de la nube describe la organización local como destino. En sentido contrario, el conector de recepción o entrante acepta el tráfico según condiciones documentadas, entre ellas la identidad del certificado y el origen.

El certificado cumple una función concreta: identifica el extremo SMTP durante el protocolo de enlace TLS. Subject o Subject Alternative Name, los parámetros del conector, la cadena de certificados presentada y el nombre de host real deben coincidir. No basta con que haya un certificado válido en el almacén de certificados si el servicio de transporte presenta otro.

Por ello, los expertos verifican ambos sentidos por separado. El sentido A puede funcionar mientras que el sentido B falla debido a otro conector, destino DNS o nombre de certificado.

## Enrutamiento de destinatarios y dominios compartidos

Un conector funcional aún no indica qué mensajes lo utilizan. Esta decisión comienza con el objeto de destinatario. Un buzón remoto local apunta a la nube. Un objeto de buzón local sincronizado en Exchange Online apunta de vuelta a la organización local.

Los dominios aceptados también determinan si una organización es responsable de todos los destinatarios de un dominio o puede reenviar destinatarios desconocidos. En dominios compartidos, una configuración Internal Relay solo es segura si el siguiente salto conoce o rechaza correctamente a los destinatarios desconocidos ([Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains), [Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)).

Por tanto, un objeto de buzón remoto desactualizado puede provocar un enrutamiento incorrecto a pesar de que TLS esté en buen estado. A la inversa, una Target Address correcta no ayuda si el conector de nube está desactivado. El diagnóstico relaciona el objeto y el transporte, en lugar de considerar solo un lado.

Para los expertos, son importantes los reenvíos, los contactos de correo, la expansión de grupos de distribución y las reglas de transporte. Pueden modificar la dirección original del destinatario o generar destinatarios adicionales. Cada mensaje resultante recibe su propia decisión de enrutamiento.

## Puertas de enlace de correo antes o después de Exchange Online

Muchas organizaciones complementan Hybrid con una Secure Email Gateway o una plataforma de filtrado en la nube. Esto añade al menos un salto SMTP adicional. La ruta debe dibujarse por separado para los mensajes entrantes y salientes ([Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)).

Si la puerta de enlace está antes de Exchange Online, el MX apunta a la puerta de enlace. Entonces EOP ve primero su IP de origen. Enhanced Filtering for Connectors puede incluir la información del remitente original en la evaluación de filtrado de Microsoft si el conector y las IP omitidas están correctamente configurados ([Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)).

Para la salida, debe estar claro si Exchange Online envía directamente, a través de la puerta de enlace o, con CMT, primero localmente y después a través de la puerta de enlace. Varias rutas permitidas pueden eludir políticas y generar firmas DKIM, IP de salida y registros diferentes.

El control experto consiste en un gráfico de rutas permitidas: cada flecha indica iniciador, destino, puerto, comprobación TLS, dominios permitidos, tarea de filtrado, propietario de la cola y fuente de registros. Un nombre de puerta de enlace sin estos datos aún no constituye una arquitectura.

## Seguir un mensaje de extremo a extremo

La solución de problemas comienza con un mensaje de prueba cuyo remitente, destinatario y hora se conocen. Primero se comprueba la ruta pública y, después, los eventos de cada lado de Exchange implicado.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für DNS- und SMTP-Test">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
Resolve-DnsName -Type MX example.com
Test-NetConnection mail.example.com -Port 25
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
dig MX example.com
nc -vz mail.example.com 25
```

  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) y [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) muestran el destino MX publicado. [`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) comprueban si TCP 25 es accesible desde el punto de medición. Esto todavía no demuestra que la comprobación de TLS o del conector haya tenido éxito.

A continuación se realiza el protocolo de enlace SMTP. OpenSSL puede utilizarse en ambas plataformas de administración.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für SMTP-STARTTLS-Test">
    <button type="button" role="tab" data-os-tab="windows" aria-selected="true">Windows</button>
    <button type="button" role="tab" data-os-tab="unix" aria-selected="false">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">

```powershell
openssl s_client -starttls smtp -connect mail.example.com:25 `
  -servername mail.example.com -showcerts
```

  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>

```bash
openssl s_client -starttls smtp -connect mail.example.com:25 \
  -servername mail.example.com -showcerts
```

  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) muestra la cadena de certificados, los nombres y la negociación TLS. Para un diálogo de prueba completo y autorizado es adecuado [`swaks`](https://jetmore.org/john/code/swaks/). Los mensajes de producción no se prueban con remitentes inventados libremente; la identidad de prueba y la ruta esperada se establecen de antemano.

En Exchange Online, [`Get-MessageTraceV2`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2) proporciona los eventos de la nube. Localmente, [`Get-MessageTrackingLog`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog) muestra el procesamiento en servidores Exchange y [`Get-Queue`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue) muestra los siguientes saltos en espera. Las marcas de tiempo se llevan a una zona horaria común; Internet Message ID y Network Message ID ayudan a vincular los segmentos ([Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq), [Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)).

## Patrones de error típicos sin cambios de tema

Un **error TLS** se trata primero como un problema de transporte: ¿Qué host se conectó, qué certificado presentó y qué condición de conector esperaba el extremo remoto? La sincronización de destinatarios solo es relevante si el mensaje se enruta incorrectamente después de haber sido aceptado correctamente.

En cambio, un **NDR por destinatario desconocido** lleva primero al objeto de destinatario y al tipo de dominio aceptado. Solo si el objeto es correcto se comprueba si la ruta elegida lo transporta al lado correcto.

Una **cola en espera** requiere el siguiente salto, la hora de reintento y `LastError`. El puerto abierto del host de destino solo ayuda como siguiente prueba. Un **bucle** se manifiesta mediante cabeceras `Received` repetidas, saltos o eventos de seguimiento y suele producirse cuando ambos lados reenvían mutuamente destinatarios desconocidos ([RFC 5321: Trace information and loop detection](https://datatracker.ietf.org/doc/html/rfc5321#section-6.3)).

Una **conexión que funciona solo en un sentido** no es una contradicción. Los sentidos opuestos utilizan emisores, conectores y comprobaciones de certificados diferentes. Se registran y prueban por separado.

## Seguridad, operación y cambios

SMTP híbrido abre una ruta de transporte deliberadamente permitida. Esta ruta debe limitarse a sistemas de origen y destino, certificados y dominios documentados. Los relés abiertos, los rangos de IP excesivamente amplios o los conectores que clasifican cualquier mensaje como confiable contradicen este modelo.

Los cambios de certificados se planifican como cambios de enrutamiento. Antes de la expiración, se comprueban el certificado nuevo, la asignación de servicio, la cadena presentada y la expectativa del conector en ambos lados. Después se realizan mensajes de prueba en ambas direcciones y se dispone de un plan de reversión controlado.

Para la operación continua se supervisan, como mínimo, los conectores híbridos, la expiración de certificados, el crecimiento de las colas, los errores de Message Trace, los destinos DNS y el estado de las puertas de enlace. Con Centralized Mail Transport se añade la capacidad de la ruta de salida local.

## Copia de seguridad y reconstrucción de la ruta de correo

Los mensajes SMTP permanecen durante una interrupción en las colas de los sistemas responsables respectivos. Una copia de seguridad de configuración no restaura estos mensajes en espera. Por ello, la configuración y el estado de transporte se consideran por separado.

La reconstrucción incluye los parámetros de conectores de ambos lados, dominios aceptados y remotos, reglas de transporte, entradas DNS públicas, certificados con claves privadas, configuración de la puerta de enlace y la selección de HCW. Los secretos se almacenan de forma protegida; las exportaciones legibles documentan la estructura y las dependencias.

Tras una reconstrucción, no se prueba solo un puerto. Un mensaje marcado recorre la ruta esperada en cada dirección. Message Trace, los registros de seguimiento locales, los registros de la puerta de enlace y el buzón de destino confirman cada salto. Solo esta aceptación de extremo a extremo demuestra que el enrutamiento y el filtrado vuelven a ser correctos.

## Evolución técnica y trade-offs

Hybrid Configuration Wizard automatizó, a lo largo de varias generaciones de Exchange, la conexión con Exchange Online. El transporte siguió siendo SMTP/TLS, mientras los conectores de nube, los certificados compatibles y las opciones de enrutamiento continuaron evolucionando ([Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)).

El transporte directo de Internet mantiene las rutas cortas y utiliza cada plataforma allí donde se encuentra el buzón. Centralized Mail Transport centraliza el control, pero hace que el correo en la nube dependa de la salida local. Una puerta de enlace de terceros añade filtrado o cifrado especializado, pero aumenta el número de saltos y fuentes de registros. La elección correcta se deriva de un requisito demostrable, no del deseo de que en el diagrama todo pase por la misma caja.

## Fuentes

- [Microsoft Learn – Transport routing in Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/transport-routing)
- [Microsoft Learn – Exchange hybrid deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Microsoft Learn – Hybrid Configuration Wizard](https://learn.microsoft.com/en-us/exchange/hybrid-configuration-wizard)
- [Microsoft Learn – Hybrid deployment prerequisites](https://learn.microsoft.com/en-us/exchange/hybrid-deployment-prerequisites)
- [Microsoft Learn – Hybrid mail flow](https://learn.microsoft.com/en-us/exchange/hybrid-deployment/hybrid-mail-flow)
- [Microsoft Learn – Enable-RemoteMailbox](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-remotemailbox)
- [Microsoft Learn – Set up connectors to route mail](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/set-up-connectors-to-route-mail)
- [Microsoft Learn – Connectors on Exchange servers](https://learn.microsoft.com/en-us/exchange/mail-flow/connectors/connectors)
- [Microsoft Learn – Accepted domains in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-accepted-domains/manage-accepted-domains)
- [Microsoft Learn – Accepted domains in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/accepted-domains/accepted-domains)
- [Microsoft Learn – Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud)
- [Microsoft Learn – Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors)
- [Microsoft Learn – Trace an email message](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/trace-an-email-message)
- [Microsoft Learn – Message Trace FAQ](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-faq)
- [Microsoft Learn – Message tracking](https://learn.microsoft.com/en-us/exchange/mail-flow/transport-logs/message-tracking)
- [Microsoft Learn – Queues in Exchange Server](https://learn.microsoft.com/en-us/exchange/mail-flow/queues/queues)
- [Microsoft Learn – Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2)
- [Microsoft Learn – Get-MessageTrackingLog](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetrackinglog)
- [Microsoft Learn – Get-Queue](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-queue)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig manual](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc(1)](https://man.openbsd.org/nc)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Swaks – SMTP test tool](https://jetmore.org/john/code/swaks/)
- [RFC 5321 – Trace information and loop detection](https://datatracker.ietf.org/doc/html/rfc5321#section-6.3)
