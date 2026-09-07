---
title: "Ghost Sender en Exchange Online: un registro MX no es un firewall"
navTitle: "Ghost Sender"
description: "La entrega directa a Exchange Online evita una puerta de enlace previa si el tenant no la bloquea expresamente. El riesgo es real; la causa es una configuración incompleta del flujo de correo."
date: "2026-07-15"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min de lectura"
themen:
  - microsoft-365-exchange
slug: "ghost-sender-en-exchange-online-un-registro-mx-no-es-un-firewall"
image: "../images/ghost-admin.png"
translationOf: "ghost-sender-exchange-online-nebeneingang"
translationId: article-d8dc8d1da6379d67
translationReview: required
translationSourceHash: 6a500f1ed53a180322afb3c86e44376100d68659eeb55ffae35937ab434c6b61
translatedAt: 2026-09-05T07:49:19.310Z
url: https://rafaelpfister.ch/es/blog/ghost-sender-en-exchange-online-un-registro-mx-no-es-un-firewall
translationModel: gpt-5.6-terra
---

# Ghost Sender en Exchange Online: un registro MX no es un firewall

![Un administrador fantasma mantiene abierta, en el centro de datos, la puerta junto a la puerta de seguridad, mientras los correos electrónicos llegan directamente al buzón sin pasar por el filtro.](../images/ghost-admin.png)

La posibilidad de ataque descrita por InfoGuard Labs como «Ghost Sender» es real: un atacante puede eludir una puerta de enlace de correo electrónico previa y entregar directamente a Exchange Online. Sin embargo, el requisito es que el tenant siga aceptando esta vía directa. No se trata de una vulnerabilidad universal de Exchange Online, sino de una topología de flujo de correo protegida de forma incompleta.

Un agente de transferencia de correo que gestiona buzones para un dominio acepta, por principio, conexiones SMTP desde Internet. El registro MX indica a los remitentes legítimos la vía de entrega deseada. No es ni una regla de firewall ni una lista de acceso, y no impide a nadie dirigirse directamente a un punto de conexión conocido de Exchange Online.

## Lo que «Ghost Sender» muestra realmente

El [escenario descrito por InfoGuard Labs](https://labs.infoguard.ch/posts/ghost-sender/) es el siguiente:

1. Una organización opera sus buzones en Exchange Online.
2. El registro MX público apunta a una Secure Email Gateway previa.
3. El punto de conexión de Exchange Online en `*.mail.protection.outlook.com` sigue siendo accesible directamente desde Internet.
4. El administrador no ha restringido Exchange Online de modo que solo la puerta de enlace previa pueda entregar allí.
5. Un atacante ignora el registro MX y entrega su mensaje directamente a Exchange Online.

La vía prevista es, por tanto:

```text
Internet -> Drittanbieter-Filter -> Exchange Online -> Postfach
```

Sin embargo, ha quedado abierta esta vía:

```text
Angreifer -> Exchange Online -> Postfach
```

Esta es una configuración incorrecta que debe tomarse en serio. El filtro previo puede eludirse por esta vía; la suplantación de remitentes, el phishing y el fraude del CEO se ven considerablemente facilitados. InfoGuard merece reconocimiento por visibilizar el problema, investigar su extensión y publicar una prueba fácil de usar.

Pero ¿dónde está exactamente el fallo de producto?

La dramatización mediática tampoco ayuda mucho a contextualizarlo. [Heise titula que Exchange Online deja pasar correos electrónicos falsificados «sin problemas»](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html), aunque solo se ven afectadas determinadas configuraciones de terceros e híbridas reforzadas de forma incompleta. [Crow in the Cloud](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) lo formula con mucha más precisión: no es una brecha de seguridad en sentido estricto, sino un problema de diseño y configuración.

## «An MTA is doing MTA-Things»

Cada tenant de Exchange Online cuenta con un punto de conexión SMTP público. Este punto de conexión no es un secreto ni debe serlo. Microsoft explica que Exchange Online acepta de forma predeterminada mensajes dirigidos directamente a buzones alojados allí: [simplemente es así como funciona el correo electrónico](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865).

También [SMTP describe el registro MX como un mecanismo para determinar el sistema de destino habitual](https://www.rfc-editor.org/rfc/rfc5321.html#section-5.1). De ello no se deriva ninguna obligación para el servidor de destino de rechazar conexiones a través de cualquier otro host accesible. Un atacante no tiene por qué seguir la ruta señalizada. Si otro MTA es accesible, conoce el dominio destinatario y acepta el mensaje, será probado, de forma muy similar a como los spammers llevan décadas intentando dirigirse a sistemas MX de respaldo peor protegidos.

Quien antepone un filtro de terceros modifica la topología estándar. De «Exchange Online es mi puerta de enlace de correo de Internet» se pasa a «solo mi puerta de enlace de terceros puede transferir correo de Internet a Exchange Online». Esta nueva `Trust-Border` no surge de una entrada DNS. Debe imponerse expresamente en el sistema receptor.

Microsoft documenta precisamente esto: con un MX externo debe crearse un conector entrante del tipo `Partner` que, para `SenderDomains *`, acepte únicamente el certificado o las direcciones IP de origen del servicio previo. Los mensajes entregados directamente sin pasar por la puerta de enlace se rechazan entonces. Así figura literalmente en la guía de Microsoft [«Manage mail flow using a third-party cloud service with Exchange Online»](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud#best-practices-for-using-a-third-party-cloud-filtering-service-with-microsoft-365-or-office-365).

Frank Carius también describe detalladamente esta «entrada lateral» en la [MSXFAQ](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm).

## SPF, DKIM y DMARC no son porteros

InfoGuard muestra mensajes en los que SPF, DKIM y DMARC fallan y que, aun así, llegan al buzón. Parece espectacular, pero no es un «bypass» criptográfico de estos mecanismos. Los correos no pasan con éxito. Entregan `fail`. Lo decisivo es qué acción local deriva el sistema receptor de ese resultado.

SPF comprueba si un sistema puede enviar para el remitente del sobre. DKIM comprueba una firma. DMARC vincula estos resultados con el dominio visible del remitente y publica un tratamiento deseado. Incluso el actual [estándar DMARC RFC 9989](https://www.rfc-editor.org/rfc/rfc9989.html#section-1) establece expresamente que el receptor puede tener en cuenta este tratamiento deseado, pero no está obligado a hacerlo. DMARC es una señal importante, pero no un control de acceso de red.

Con una puerta de enlace previa se añade que Exchange Online ve primero la dirección IP de esta puerta de enlace, y no la del remitente original. Para ello existe [Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): reconstruye el origen original y mejora las evaluaciones de SPF, DKIM, DMARC, anti-spoofing y anti-phishing. Sin embargo, Enhanced Filtering tampoco es una cerradura. No sustituye al conector de socio restrictivo.

La configuración incorrecta se hace especialmente evidente cuando un administrador debilita o elimina la comprobación de EOP mediante un bypass de SCL porque, al fin y al cabo, el producto previo ya debe filtrar, pero al mismo tiempo deja abierta la entrega directa desde Internet. Entonces no le han «eludido» un mecanismo de protección, sino que ha previsto conscientemente que una de las dos entradas ya no tenga una protección efectiva.

Se puede criticar con razón a Microsoft si un mensaje con un error de autenticación claramente visible llega a la bandeja de entrada sin advertencia. Se puede criticar la semántica de los tipos de conector, la documentación y la ausencia de advertencias en Configuration Analyzer. Todos esos son puntos legítimos. Sin embargo, la existencia de un punto de conexión SMTP públicamente accesible no es una brecha de seguridad.

## «Direct Send» no equivale a «entrega directa»

En el debate se mezclan dos cosas:

- **Direct Send** designa en Microsoft mensajes anónimos cuyo remitente del sobre (`5321.MailFrom`) utiliza un dominio aceptado propio del tenant.
- **Entrega directa a Exchange Online** designa en general un mensaje SMTP que ignora el MX de terceros publicado y se entrega directamente al punto de conexión de Exchange. El remitente también puede utilizar cualquier dominio externo.

Para Direct Send hay un interruptor específico:

```powershell
Set-OrganizationConfig -RejectDirectSend $true
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-RejectDirectSend $true` | Rechaza entregas directas anónimas cuyo remitente del sobre utiliza un dominio aceptado del tenant |

</details>

El interruptor es útil si no se necesita Direct Send. Evita la suplantación de dominios internos por esta vía. Sin embargo, no cierra toda la entrada lateral para remitentes externos arbitrarios. Microsoft describe el ámbito exacto en la [documentación del cmdlet para `RejectDirectSend`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-organizationconfig?view=exchange-ps#-rejectdirectsend). Quien quiera impedir por completo «Ghost Sender» sigue necesitando una restricción de acceso mediante un conector de socio o una regla de flujo de correo adecuada.

## ¿De verdad Microsoft debe hacerlo todo por el administrador?

No. Quien incorpora un filtro de correo adicional a una cadena de transporte productiva asume la responsabilidad de esa cadena de transporte.

El proveedor no puede adivinar de forma fiable si, además del MX externo, los escáneres, los equipos multifunción, los servicios SaaS, los servidores híbridos, los relés de socios u otros sistemas legítimos también deben enviar directamente a Exchange Online. Un «el MX apunta a otro sitio, así que bloqueo todo lo demás» automático interrumpiría flujos de correo deseados en numerosos entornos reales. Por eso el administrador debe definir explícitamente el límite de confianza deseado.

Aun así, Microsoft puede facilitarlo a los responsables. Un buen Configuration Analyzer debería detectar un MX externo sin conector de socio restrictivo y advertirlo claramente. El asistente de configuración podría explicar que un conector del tipo «su organización» identifica las conexiones adecuadas, pero no rechaza automáticamente las conexiones inadecuadas. También serían bienvenidos interruptores secure-by-default y mejores informes operativos.

Eso sería un refuerzo de producto sensato. Pero no cambia la clasificación técnica: una topología especial insegura sigue siendo una configuración insegura y no se convierte en un zero-day únicamente por estar muy extendida.

## Cómo cerrar la entrada lateral

Para entornos con un filtro previo, al menos estos puntos deben figurar en la lista de verificación:

1. **Documentar por completo el flujo de correo.** ¿Qué sistemas pueden entregar realmente a Exchange Online? Esto incluye también rutas híbridas, de aplicaciones y de emergencia.
2. **Configurar un conector de socio restrictivo.** Utilice `SenderDomains *` y limite la entrega a un certificado (preferiblemente) o a rangos de IP de origen mantenidos. Un conector del tipo `OnPremises` o «su organización» no impone este efecto de denegación predeterminada (véase, por ejemplo, también: [Enrutamiento de correo entre Apache James y Exchange Online](/blog/totemomail-m365)).
3. **Configurar correctamente Enhanced Filtering.** Si EOP debe seguir filtrando, la IP original y la información del remitente deben reconstruirse correctamente. Los bypasses globales de SCL-`-1` deben revisarse de forma crítica.
4. **Desactivar Direct Send si no se utiliza.** Antes, compruebe mediante Message Trace o los informes disponibles si los escáneres o las aplicaciones dependen de ello.
5. **No cambiar a ciegas.** Pruebe y supervise posteriormente los rangos IP de la puerta de enlace, los cambios de certificado, el flujo de correo híbrido y las rutas especiales de `onmicrosoft.com`-, Teams y otras.

Un ejemplo simplificado de la variante basada en IP es:

```powershell
New-InboundConnector `
  -Name "Only from upstream mail gateway" `
  -ConnectorType Partner `
  -SenderDomains * `
  -RestrictDomainsToIPAddresses $true `
  -SenderIpAddresses <IP-Bereiche-des-Gateways> `
  -RequireTls $true
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Name` | Nombre para mostrar del nuevo conector entrante |
| `-ConnectorType Partner` | Clase de conector para sistemas de socios externos; solo este tipo impone el rechazo de conexiones inadecuadas |
| `-SenderDomains *` | El conector se aplica al correo de todos los dominios remitentes |
| `-RestrictDomainsToIPAddresses $true` | Activa el bloqueo: el correo de los dominios indicados solo se acepta desde las direcciones de `-SenderIpAddresses` |
| `-SenderIpAddresses` | Las direcciones IP o rangos de origen permitidos de la puerta de enlace previa |
| `-RequireTls $true` | Exige cifrado TLS para conexiones a través de este conector |

</details>

Cuando sea posible, la vinculación mediante certificado es preferible a una lista de permitidos por IP. Los cambios deben realizarse primero en una prueba controlada, pues una lista de permitidos errónea convierte rápidamente la entrada lateral abierta en una interrupción total del correo.

## La sencilla autoprueba

La prueba mostrada por InfoGuard (y MSXFAQ) es útil:

```powershell
Send-MailMessage `
  -SmtpServer <tenantname>.mail.protection.outlook.com `
  -To admin@<tenantdomain> `
  -From noreply@example.com `
  -Subject "EXO Nebeneingang" `
  -Body "Testmail direkt zum Tenant"
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-SmtpServer` | Host de destino: el punto de conexión público de Exchange Online del tenant, deliberadamente sin pasar por el MX |
| `-To` | Dirección del destinatario en el tenant que se desea probar |
| `-From` | Cualquier dirección de remitente externa; precisamente esto es lo que la entrada lateral ya no debería aceptar |
| `-Subject` | Línea de asunto, para localizarla de nuevo en Message Trace |
| `-Body` | Texto del mensaje |

</details>

Con un conector de socio correctamente restringido, cabe esperar un rechazo SMTP como `5.7.51 TenantInboundAttribution; Rejecting`. Una regla de transporte alternativa puede aceptar primero el mensaje y trasladarlo después a cuarentena; por ello, además de la respuesta SMTP, deben comprobarse Message Trace, la cuarentena y el buzón. `Send-MailMessage` (obsoleto) sirve aquí únicamente como ilustración fácilmente comprensible. Cualquier herramienta de prueba SMTP controlada cumple el mismo propósito.

## Una prueba útil con una etiqueta engañosa

«Ghost Sender» no es un nuevo exploit SMTP. Es un nombre pegadizo para una entrada lateral abierta cuya protección Microsoft documenta desde hace tiempo y que el administrador ha dejado abierta.

Lo irónico es que InfoGuard califica el problema en su propio artículo como «widespread and systematic misconfiguration» y concluye con la frase «Ghost-Sender is a misconfiguration». El Security Response Center de Microsoft también clasificó inicialmente el informe como no correspondiente a una brecha de seguridad. Por tanto, los hechos están presentes en el artículo: lamentablemente, solo el título, el correo de prueba y la marca «Vulnerability» sugieren una interpretación más dramática.

La parte útil de la publicación es la llamada de atención: aparentemente, muchas empresas no han asegurado correctamente su flujo de correo. La parte problemática es la afirmación de que Exchange Online tiene una brecha de seguridad universal por ello. No: Exchange Online se comporta inicialmente aquí como un MTA. Se vuelve inseguro por un límite de confianza cuya configuración no se ha completado.

¿Hay que quitárselo todo al administrador? No. Pero al parecer hay que recordar una y otra vez que el enrutamiento DNS no sustituye al control de acceso.

## Fuentes

1.  [InfoGuard Labs: Ghost-Sender – Universal Email Spoofing against Exchange Online](https://labs.infoguard.ch/posts/ghost-sender/): La investigación original, incluido el análisis de difusión y la conclusión de los propios autores: «Ghost-Sender is a misconfiguration».

2.  [Ghost Sender: Exchange Online Mail Spoofing Tester](https://ghost-sender.com/): La prueba en línea publicada por InfoGuard para comprobar si el propio tenant tiene la entrada lateral abierta.

3.  [MSXFAQ: Exchange Online como entrada lateral para la recepción de correo](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm): La valoración de Frank Carius: no es un error en Exchange Online, sino una configuración incorrecta del administrador.

4.  [Microsoft: Direct Send vs sending directly to an Exchange Online tenant](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865): Microsoft explica que la aceptación directa de correo a buzones alojados es la forma en que funciona el correo electrónico, y delimita Direct Send.

5.  [Microsoft Learn: Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): La guía oficial con su propio paso para el conector de socio restrictivo con MX externo.

6.  [Microsoft Learn: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): Reconstruye el origen original del remitente detrás de una puerta de enlace; mejora la evaluación, pero no sustituye al conector.

7.  [Heise: Ghost-Sender – Exchange Online deja pasar correos electrónicos falsificados sin problemas](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html): Ejemplo de cobertura sensacionalista que generaliza solo determinadas configuraciones incorrectas.

8.  [Crow in the Cloud: Los fantasmas que no invoqué](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/): Acertada clasificación como problema de diseño y configuración, junto con medidas de protección.

9.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321.html): Describe el registro MX como mecanismo para determinar el sistema de destino habitual, no como control de acceso.

10.  [RFC 9989: DMARC](https://www.rfc-editor.org/rfc/rfc9989.html): Establece que el receptor puede tener en cuenta el tratamiento DMARC publicado, pero no tiene que hacerlo.

---

## ¿Es seguro su flujo de correo?

¿No está seguro de si su tenant de Exchange Online también tiene una entrada lateral abierta? **adeptio** revisa todo su flujo de correo: desde registros MX, conectores y puertas de enlace de terceros hasta EOP, SPF, DKIM, DMARC y Direct Send. De forma práctica, independiente y con recomendaciones concretas.

Quien desee revisar su flujo de correo o protegerlo adecuadamente puede concertar sin compromiso una consulta:

**[Reservar una consulta con adeptio](https://outlook.office.com/bookwithme/user/b4d64d6bdbca4b489074d459cd30b50c@adeptio.ch/meetingtype/3Wgk7rXJfk261852Hyovkg2?anonymous&ismsaljsauthenabled&ep=mlink)**  
[adeptio.ch](https://adeptio.ch/)
