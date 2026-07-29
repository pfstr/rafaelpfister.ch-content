---
title: "Ghost Sender en Exchange Online: un registro MX no es un firewall"
navTitle: "Ghost Sender"
description: "La entrega directa a Exchange Online elude una pasarela previa si el tenant no la bloquea expresamente. El riesgo es real; la causa es una configuración incompleta del flujo de correo."
date: "2026-07-15"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min de lectura"
themen:
  - microsoft-365-exchange
slug: "ghost-sender-en-exchange-online-un-registro-mx-no-es-un-firewall"
image: "../images/ghost-admin.png"
translationOf: "ghost-sender-exchange-online-nebeneingang"
url: "https://rafaelpfister.ch/es/blog/ghost-sender-en-exchange-online-un-registro-mx-no-es-un-firewall"
translationId: article-d8dc8d1da6379d67
translationReview: automatic
translationSourceHash: fc228adeba2a4ea46f6b36d20946d0aeb5c30f485b32da965e52168d2806a689
translatedAt: 2026-07-29T12:29:38.952Z
---

# Ghost Sender en Exchange Online: un registro MX no es un firewall

![Un administrador fantasma mantiene abierta la puerta junto a la puerta de seguridad en el centro de datos, mientras los correos electrónicos pasan directamente al buzón sin pasar por el filtro.](../images/ghost-admin.png)

La posibilidad de ataque descrita por InfoGuard Labs como «Ghost Sender» es real: un atacante puede eludir una pasarela de correo electrónico previa y entregar directamente a Exchange Online. Sin embargo, el requisito es que el tenant siga aceptando esta vía directa. No se trata de una vulnerabilidad universal de Exchange Online, sino de una topología de flujo de correo protegida de forma incompleta.

Un Mail Transfer Agent que gestiona buzones para un dominio acepta, en principio, conexiones SMTP desde Internet. El registro MX indica a los remitentes legítimos la ruta de entrega deseada. No es ni una regla de firewall ni una lista de acceso, y no impide a nadie dirigirse directamente a un punto de conexión conocido de Exchange Online.

## Lo que «Ghost Sender» muestra realmente

El escenario [descrito por InfoGuard Labs](https://labs.infoguard.ch/posts/ghost-sender/) es el siguiente:

1. Una organización aloja sus buzones en Exchange Online.
2. El registro MX público apunta a una Secure Email Gateway previa.
3. El punto de conexión de Exchange Online en `*.mail.protection.outlook.com` sigue siendo accesible directamente desde Internet.
4. El administrador no ha restringido Exchange Online para que solo la pasarela previa pueda entregar allí.
5. Un atacante ignora el registro MX y entrega su mensaje directamente a Exchange Online.

Por tanto, la ruta prevista es:

```text
Internet -> Drittanbieter-Filter -> Exchange Online -> Postfach
```

Sin embargo, esta ruta permanece abierta:

```text
Angreifer -> Exchange Online -> Postfach
```

Se trata de una configuración errónea que debe tomarse en serio. El filtro previo puede eludirse por esta vía; la suplantación de remitentes, el phishing y el fraude del CEO resultan así considerablemente más fáciles. InfoGuard merece reconocimiento por visibilizar el problema, investigar su difusión y publicar una prueba fácil de usar.

Pero ¿dónde está exactamente el fallo de producto?

La exageración mediática tampoco ayuda mucho a contextualizarlo. [Heise titula que Exchange Online deja pasar correos falsificados «sin problemas»](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html), aunque solo están afectadas determinadas configuraciones de terceros e híbridas que no se han reforzado completamente. [Crow in the Cloud](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/) lo formula con mucha más precisión: no es una brecha de seguridad en sentido estricto, sino un problema de diseño y configuración.

## «An MTA is doing MTA-Things»

Cada tenant de Exchange Online tiene un punto de conexión SMTP público. Este punto de conexión no es un secreto ni debe serlo. Microsoft explica que Exchange Online acepta de forma predeterminada los mensajes dirigidos directamente a buzones alojados allí: [simplemente es así como funciona el correo electrónico](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865).

También [el propio SMTP describe el registro MX como un mecanismo para determinar el sistema de destino habitual](https://www.rfc-editor.org/rfc/rfc5321.html#section-5.1). De ello no se deriva obligación alguna para el servidor de destino de rechazar conexiones a través de cualquier otro host accesible. Un atacante no tiene que seguir la ruta señalizada. Si otro MTA es accesible, conoce el dominio destinatario y acepta el mensaje, se probará esa vía, de manera muy similar a como los spammers llevan décadas intentando contactar sistemas MX de respaldo menos protegidos.

Quien coloca un filtro de terceros delante modifica la topología estándar. «Exchange Online es mi pasarela de correo de Internet» pasa a ser «solo mi pasarela de terceros puede entregar correo de Internet a Exchange Online». Este nuevo `Trust-Border` no surge mediante una entrada DNS. Debe imponerse expresamente en el sistema receptor.

Microsoft documenta precisamente esto: con un MX externo debe crearse un conector entrante de tipo `Partner` que, para `SenderDomains *`, acepte únicamente el certificado o las direcciones IP de origen del servicio previo. Los mensajes entregados directamente sin pasar por la pasarela se rechazan entonces. Así figura literalmente en la guía de Microsoft [«Manage mail flow using a third-party cloud service with Exchange Online»](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud#best-practices-for-using-a-third-party-cloud-filtering-service-with-microsoft-365-or-office-365).

Frank Carius también describe detalladamente esta «entrada lateral» en [MSXFAQ](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm).

## SPF, DKIM y DMARC no son porteros

InfoGuard muestra mensajes en los que SPF, DKIM y DMARC fallan y que, aun así, llegan al buzón. Parece espectacular, pero no es un «bypass» criptográfico de estos mecanismos. Los correos no superan las comprobaciones correctamente. Entregan `fail`. Lo decisivo es qué acción local deriva el sistema receptor de este resultado.

SPF comprueba si un sistema puede enviar para el remitente del sobre. DKIM comprueba una firma. DMARC relaciona estos resultados con el dominio visible del remitente y publica un tratamiento deseado. Incluso el actual [estándar DMARC RFC 9989](https://www.rfc-editor.org/rfc/rfc9989.html#section-1) establece expresamente que el destinatario puede tener en cuenta este tratamiento deseado, pero no está obligado a ello. DMARC es una señal importante, pero no un control de acceso de red.

Con una pasarela previa se añade que Exchange Online ve primero la dirección IP de esta pasarela y no la del remitente original. Para ello existe [Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): reconstruye el origen original y mejora las evaluaciones de SPF, DKIM, DMARC, anti-spoofing y anti-phishing. Sin embargo, Enhanced Filtering tampoco es una cerradura. No sustituye al conector de partner restrictivo.

La configuración errónea resulta especialmente evidente cuando un administrador debilita o elimina por completo la comprobación de EOP mediante un bypass de SCL, porque se supone que el producto previo ya debe filtrar, pero al mismo tiempo deja abierta la entrega directa desde Internet. En ese caso, no se le ha «eludido» un mecanismo de protección, sino que ha decidido no prever ya una protección eficaz para una de las dos entradas.

Se puede criticar a Microsoft si un mensaje con un fallo de autenticación claramente visible llega a la bandeja de entrada sin advertencia. Se puede criticar la semántica de los tipos de conectores, la documentación y la falta de advertencias en Configuration Analyzer. Todos ellos son puntos legítimos. Sin embargo, la existencia de un punto de conexión SMTP accesible públicamente no es una vulnerabilidad de seguridad.

## «Direct Send» no es lo mismo que «entrega directa»

En el debate se mezclan dos cosas:

- **Direct Send** designa en Microsoft mensajes anónimos cuyo remitente del sobre (`5321.MailFrom`) utiliza un dominio aceptado propio del tenant.
- **Entrega directa a Exchange Online** designa, en general, un mensaje SMTP que ignora el MX de terceros publicado y se entrega directamente al punto de conexión de Exchange. El remitente también puede utilizar cualquier dominio externo.

La opción

```powershell
Set-OrganizationConfig -RejectDirectSend $true
```

es útil si no se necesita Direct Send. Evita la suplantación de dominios internos por esta vía. Sin embargo, no cierra toda la entrada lateral para remitentes externos arbitrarios. Microsoft describe el ámbito exacto en la [documentación del cmdlet de `RejectDirectSend`](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-organizationconfig?view=exchange-ps#-rejectdirectsend). Quien quiera impedir «Ghost Sender» por completo sigue necesitando la restricción de acceso mediante un conector de partner o una regla de flujo de correo adecuada.

## ¿De verdad Microsoft debe encargarse de todo por el administrador?

No. Quien integra un filtro de correo adicional en una cadena de transporte productiva asume la responsabilidad de esa cadena de transporte.

El proveedor no puede adivinar de forma fiable si, además del MX externo, escáneres, dispositivos multifunción, servicios SaaS, servidores híbridos, relés de partners u otros sistemas legítimos deben enviar directamente a Exchange Online. Un bloqueo automático de «el MX apunta a otro lugar, así que bloqueo todo lo demás» interrumpiría flujos de correo deseados en numerosos entornos reales. Por ello, el administrador debe definir explícitamente el límite de confianza deseado.

Aun así, Microsoft debería facilitarlo a los responsables. Un buen Configuration Analyzer debería detectar un MX externo sin un conector de partner restrictivo y advertirlo claramente. El asistente de configuración podría explicar que un conector de tipo «Su organización» identifica conexiones adecuadas, pero no rechaza automáticamente las conexiones inadecuadas. También serían bienvenidos interruptores seguros por defecto e informes operativos mejores.

Eso sería un refuerzo de producto razonable. Pero no cambia la clasificación técnica: una topología especial insegura sigue siendo una configuración insegura y no se convierte en un zero-day solo por su amplia difusión.

## Cómo cerrar la entrada lateral

Para entornos con un filtro previo, al menos estos puntos deben figurar en la lista de verificación:

1. **Documentar completamente el flujo de correo.** ¿Qué sistemas pueden realmente entregar a Exchange Online? Esto incluye también rutas híbridas, de aplicaciones y de emergencia.
2. **Configurar un conector de partner restrictivo.** Utilizar `SenderDomains *` y limitar la entrega a un certificado (preferiblemente) o a rangos de IP de origen mantenidos. Un conector de tipo `OnPremises` o «Su organización» no impone este efecto de denegación por defecto (véase, por ejemplo, [Enrutamiento de correo entre Apache James y Exchange Online](/blog/totemomail-m365)).
3. **Configurar correctamente Enhanced Filtering.** Si EOP debe seguir filtrando, la IP original y la información del remitente deben reconstruirse correctamente. Los bypasses generales de SCL-`-1` deben revisarse de forma crítica.
4. **Desactivar Direct Send si no se utiliza.** Antes, comprobar mediante Message Trace o los informes disponibles si los escáneres o las aplicaciones dependen de ello.
5. **No cambiar a ciegas.** Probar y supervisar después los rangos de IP de la pasarela, los cambios de certificado, el flujo de correo híbrido y las rutas especiales de `onmicrosoft.com`, Teams y otras.

Un ejemplo simplificado de la variante basada en IP es:

```powershell
New-InboundConnector `
  -Name "Solo desde la pasarela de correo previa" `
  -ConnectorType Partner `
  -SenderDomains * `
  -RestrictDomainsToIPAddresses $true `
  -SenderIpAddresses <Rangos-IP-de-la-pasarela> `
  -RequireTls $true
```

Cuando sea posible, debe preferirse la vinculación por certificado a una lista de permitidos de IP. Los cambios deben probarse primero en un entorno controlado, pues una lista de permitidos incorrecta convierte rápidamente la entrada lateral abierta en una interrupción completa del correo.

## La sencilla autoprueba

La prueba mostrada por InfoGuard (y MSXFAQ) es útil:

```powershell
Send-MailMessage `
  -SmtpServer <nombredeltenant>.mail.protection.outlook.com `
  -To admin@<dominiodeltenant> `
  -From noreply@example.com `
  -Subject "Entrada lateral de EXO" `
  -Body "Correo de prueba directamente al tenant"
```

Con un conector de partner correctamente restringido, cabe esperar un rechazo SMTP como `5.7.51 TenantInboundAttribution; Rejecting`. Una regla de transporte alternativa puede aceptar primero el mensaje y moverlo después a cuarentena; por ello, además de la respuesta SMTP, deben comprobarse Message Trace, la cuarentena y el buzón. `Send-MailMessage` (obsoleto) sirve aquí únicamente como ilustración fácil de entender. Cualquier herramienta de prueba SMTP controlada cumple el mismo propósito.

## Una prueba útil con una etiqueta engañosa

«Ghost Sender» no es un nuevo exploit SMTP. Es un nombre llamativo para una entrada lateral abierta cuya protección Microsoft documenta desde hace tiempo y que el administrador ha dejado abierta.

La ironía es que InfoGuard denomina el problema en su propia publicación «widespread and systematic misconfiguration» y concluye con la frase «Ghost-Sender is a misconfiguration». El Security Response Center de Microsoft tampoco clasificó inicialmente el informe como una vulnerabilidad de seguridad. Por tanto, los hechos están presentes en el artículo: solo el título, el correo de prueba y la etiqueta «Vulnerability» cuentan, por desgracia, una historia más dramática.

La parte útil de la publicación es la llamada de atención: aparentemente, muchas empresas no han asegurado correctamente su flujo de correo. La parte problemática es la afirmación de que Exchange Online tiene por ello una vulnerabilidad de seguridad universal. No: Exchange Online se comporta aquí inicialmente como un MTA. Se vuelve inseguro debido a un límite de confianza que no se ha configurado hasta el final.

¿De verdad hay que encargarse de todo por el administrador? No. Pero, al parecer, hay que recordar una y otra vez que el enrutamiento DNS no sustituye al control de acceso.

## Fuentes

1.  [InfoGuard Labs: Ghost-Sender – Universal Email Spoofing against Exchange Online](https://labs.infoguard.ch/posts/ghost-sender/): La investigación original, incluido el análisis de difusión y la conclusión propia: «Ghost-Sender is a misconfiguration».

2.  [Ghost Sender: Exchange Online Mail Spoofing Tester](https://ghost-sender.com/): La prueba en línea publicada por InfoGuard para comprobar si el propio tenant tiene la entrada lateral abierta.

3.  [MSXFAQ: Exchange Online como entrada lateral para la recepción de correo](https://www.msxfaq.de/cloud/exchangeonline/transport/exo-nebeneingang.htm): La clasificación de Frank Carius: no es un fallo de Exchange Online, sino una configuración errónea del administrador.

4.  [Microsoft: Direct Send vs sending directly to an Exchange Online tenant](https://techcommunity.microsoft.com/blog/exchange/direct-send-vs-sending-directly-to-an-exchange-online-tenant/4439865): Microsoft explica que la aceptación directa de correo para buzones alojados es el funcionamiento del correo electrónico y delimita Direct Send.

5.  [Microsoft Learn: Manage mail flow using a third-party cloud service](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/manage-mail-flow-using-third-party-cloud): La guía oficial con su propio paso para el conector de partner restrictivo con MX externo.

6.  [Microsoft Learn: Enhanced Filtering for Connectors](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/enhanced-filtering-for-connectors): Reconstruye el origen original del remitente detrás de una pasarela; mejora la evaluación, pero no sustituye al conector.

7.  [Heise: Ghost-Sender – Exchange Online deja pasar correos falsificados sin problemas](https://www.heise.de/news/Ghost-Sender-Exchange-Online-laesst-gefaelschte-E-Mails-anstandslos-durch-11327666.html): Ejemplo de cobertura exagerada que generaliza solo determinadas configuraciones erróneas.

8.  [Crow in the Cloud: Los fantasmas que no invoqué](https://crowinthe.cloud/die-geister-die-ich-nicht-rief-effektiver-schutz-gegen-ghost-sender-in-exchange-online/): Clasificación acertada como problema de diseño y configuración, con medidas de protección.

9.  [RFC 5321: Simple Mail Transfer Protocol](https://www.rfc-editor.org/rfc/rfc5321.html): Describe el registro MX como mecanismo para determinar el sistema de destino habitual, no como control de acceso.

10.  [RFC 9989: DMARC](https://www.rfc-editor.org/rfc/rfc9989.html): Establece que el destinatario puede tener en cuenta el tratamiento DMARC publicado, pero no está obligado a hacerlo.

---

## ¿Es seguro su flujo de correo?

¿No está seguro de si su tenant de Exchange Online también tiene una entrada lateral abierta? **adeptio** revisa todo su flujo de correo: desde registros MX, conectores y pasarelas de terceros hasta EOP, SPF, DKIM, DMARC y Direct Send. De forma práctica, independiente y con recomendaciones concretas.

Quien desee revisar o proteger correctamente su flujo de correo puede concertar una consulta sin compromiso:

**[Reservar una consulta con adeptio](https://outlook.office.com/bookwithme/user/b4d64d6bdbca4b489074d459cd30b50c@adeptio.ch/meetingtype/3Wgk7rXJfk261852Hyovkg2?anonymous&ismsaljsauthenabled&ep=mlink)**  
[adeptio.ch](https://adeptio.ch/)
