---
title: "¿Quién entrega realmente correo a su tenant? Agregar direcciones IP de entrega"
navTitle: "IPs de entrega"
description: "Un único análisis muestra qué sistemas entregan realmente correo a su tenant: conectores olvidados, aplicaciones que envían directamente y proveedores de servicios que nadie ha documentado. Incluye los obstáculos de la lógica de paginación y la interpretación."
date: "2026-08-11"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "12 min de lectura"
themen:
  - microsoft-365-exchange
  - smtp-mailflow
  - exchange-onprem-hybrid
hauptthema: "microsoft-365-exchange"
produkte:
  - "exchange-online"
  - "exchange-hybrid"
protokolle:
  - "smtp"
  - "powershell"
  - "haertung"
related:
  - exchange-message-tracking-und-receive-connectoren-analysieren
  - ghost-sender-exchange-online-nebeneingang
  - mailflow-fehlersuche-kontrollierte-experimente
slug: "quien-entrega-realmente-correo-a-su-tenant-agregar-direcciones-ip-de-entrega"
translationId: "article-5879cc0eb17ed951"
draft: false
translationOf: einliefernde-ip-adressen-aggregieren
url: https://rafaelpfister.ch/es/blog/quien-entrega-realmente-correo-a-su-tenant-agregar-direcciones-ip-de-entrega
translationSourceHash: 9dc48329a06945f705380eb3db428efb548f0c36a1fe3c4f2fb7de1185fee879
translationModel: gpt-5.6-terra
translatedAt: 2026-08-12T05:13:05.030Z
translationReview: automatic
---

# ¿Quién entrega realmente correo a su tenant? Agregar direcciones IP de entrega

Casi ningún entorno de correo sabe ya por completo quién le entrega mensajes. Con los años se acumulan conectores de migraciones, aplicaciones que envían directamente, proveedores de servicios cuyos contratos expiraron hace tiempo y configuraciones de prueba que nunca se desmontaron. Mientras el correo fluya, nadie lo nota.

Un único análisis aporta claridad: la agrupación de todos los mensajes entrantes por su dirección IP de origen. Se realiza en dos minutos y la lista de resultados suele ser sorprendente. Este artículo muestra la consulta, explica cómo obtenerla **completa** y, sobre todo, cómo interpretar correctamente las cifras. Porque la interpretación es la parte más difícil.

## Por qué vale la pena

La lista responde a cuatro preguntas que, de otro modo, resultan laboriosas de aclarar individualmente. ¿Qué sistemas envían realmente a su tenant? ¿Todo pasa por las vías que ha documentado o existe una segunda entrada? ¿Se sigue utilizando un conector que quiere retirar? Y: ¿una aplicación envía directamente al servicio sin pasar por su gateway, evitando así su filtrado?

La última pregunta, en particular, es relevante para la seguridad. Quien entrega directamente no solo evita el filtrado, sino a menudo también el registro en el que quiere confiar en caso de incidente.

## La consulta

En el tenant, agrupe el seguimiento de mensajes por `FromIP`:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) `
    -EndDate (Get-Date) `
    -ResultSize 5000 |
  Group-Object FromIP |
  Sort-Object Count -Descending |
  Format-Table Count, Name -AutoSize
```

Una salida típica:

```text
Count Name
----- ----
 1771 255.255.255.255
 1649 10.0.20.23
  260 10.0.20.21
   49 2603:10a6:150:1f3::17
   46 165.225.94.87
   36 136.226.192.164
   35 147.161.246.105
   12 198.51.100.77
    3 203.0.113.9
```

Antes de sacar conclusiones, deben cumplirse dos condiciones: la lista debe estar completa y debe saber qué significan las entradas.

## Obstáculo 1: La lista casi siempre está incompleta

`Get-MessageTraceV2` devuelve los resultados por páginas, con un máximo de 5000 líneas por llamada. Con un volumen elevado, una página solo cubre una fracción de su ventana temporal. Entonces agrupa sobre una muestra y considera el resultado como el total.

Esto se reconoce por la siguiente advertencia:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

**Si aparece esta advertencia, su análisis no sirve para nada.** En particular, la ausencia de una entrada no debe interpretarse como inexistencia. Una dirección con tres mensajes al día tampoco aparecerá en una muestra.

Hay dos salidas. La sencilla: reduzca la ventana temporal hasta que la advertencia deje de aparecer. Con 5000 mensajes por hora, serán 55 minutos y no siete días. Para responder a la pregunta «qué sistemas envían realmente», una ventana corta pero completa suele ser suficiente.

El método exhaustivo recorre todas las páginas y recopila los resultados:

```powershell
$start = (Get-Date).AddHours(-24)
$ende  = Get-Date
$alle  = @()
$naechster = $null

do {
    $seite = if ($naechster) {
        Get-MessageTraceV2 -StartDate $start -EndDate $ende `
            -StartingRecipientAddress $naechster -ResultSize 5000
    } else {
        Get-MessageTraceV2 -StartDate $start -EndDate $ende -ResultSize 5000
    }

    $alle += $seite
    $naechster = if ($seite.Count -eq 5000) { $seite[-1].RecipientAddress } else { $null }
    Write-Host "Gesammelt: $($alle.Count)"
} while ($naechster)

$alle | Group-Object FromIP | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

Para 24 horas en un entorno mediano, cuente con varios minutos de ejecución. Para un inventario puntual, es tiempo bien invertido.

## Obstáculo 2: Las cifras no significan lo que parecen significar

La lista de resultados contiene cuatro tipos de entradas completamente distintos, y quien los mezcle sacará conclusiones erróneas.

**`255.255.255.255` no representa un sistema.** Este valor aparece cuando no hubo una conexión SMTP entrante desde el exterior para el mensaje. Esto afecta a los mensajes generados por el propio servicio: informes de diario, notificaciones de no entrega, respuestas automáticas de ausencia y mensajes entre buzones del mismo tenant. En casi todos los entornos, es la partida más grande y es totalmente normal. No se alarme.

**Las direcciones privadas de RFC 1918** proceden de su propia red. En entornos híbridos verá aquí los servidores de transporte locales, ya que su dirección interna se conserva al entregar al servicio. Son las cifras grandes de la lista y, por regla general, la vía principal esperada.

**Las direcciones de servicios de seguridad y filtrado** se identifican por el operador, no por su valor numérico. Los proxies en la nube, las puertas de enlace de correo previas y los servicios de seguridad web aparecen con muchas direcciones adyacentes y cifras medias. Normalmente corresponden a servicios legítimos, pero deberían figurar en el manual operativo.

**Las direcciones públicas individuales con cifras bajas** son las interesantes. Ahí es precisamente donde se esconden aplicaciones olvidadas, antiguos proveedores y sistemas de los que ya nadie se acuerda.

## La resolución: de direcciones a nombres

Para todo lo que no pueda asignar de inmediato, ayuda la resolución inversa. No siempre está configurada ni siempre es fiable, pero en la mayoría de los casos proporciona la pista decisiva:

```powershell
$unbekannt = '198.51.100.77','203.0.113.9'

$unbekannt | ForEach-Object {
    $name = try { (Resolve-DnsName $_ -Type PTR -ErrorAction Stop).NameHost } catch { '(kein PTR)' }
    [pscustomobject]@{ IP = $_; Name = $name }
} | Format-Table -AutoSize
```

```text
IP            Name
--            ----
198.51.100.77 mail-out-03.newsletter-provider.example
203.0.113.9   (kein PTR)
```

La ausencia de PTR no prueba que haya algo malicioso, pero es un buen motivo para investigar más a fondo. Para esas direcciones, revise los mensajes correspondientes:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.FromIP -eq '203.0.113.9' } |
  Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize
```

El remitente y el asunto normalmente le indicarán enseguida qué aplicación hay detrás.

## La comparación: ¿qué dirección corresponde a qué conector?

Ahora llega el verdadero valor del análisis. Compare su lista de resultados con los conectores configurados:

```powershell
Get-InboundConnector |
    Format-List Name, Enabled, ConnectorType, ConnectorSource,
        @{n='SenderIPAddresses'; e={$_.SenderIPAddresses -join ', '}},
        @{n='SenderDomains';     e={$_.SenderDomains -join ', '}},
        RequireTls, TlsSenderCertificateName,
        RestrictDomainsToCertificate, CloudServicesMailEnabled, TreatMessagesAsInternal
```

Tres situaciones son reveladoras.

**Una dirección entrega correo, pero no figura en ningún conector.** Entonces el correo llega como correo normal de Internet. Esto está permitido, pero significa que esta aplicación no recibe ningún tratamiento especial y que sus mensajes están sujetos al filtrado completo. Si alguien afirma que hay un conector configurado para este sistema, claramente ya no es cierto.

**Un conector contiene direcciones desde las que no llega nada.** Es candidato a retirarse. Antes de eliminarlo, compruebe si se trata de sistemas estacionales o poco frecuentes y desactívelo primero en vez de quitarlo de inmediato.

**Un conector establece `TreatMessagesAsInternal` o `CloudServicesMailEnabled` en verdadero.** Aquí conviene mirar con atención. Ambas configuraciones hacen que los mensajes que llegan por esta vía se traten como internos de la organización. Si por ella entra correo desde Internet, se omiten comprobaciones destinadas a mensajes externos, incluida la protección contra remitentes falsificados del propio dominio. Para un conector híbrido puro, esto es correcto; para un conector por el que entregan sistemas arbitrarios, es un hallazgo.

## Lo que suele encontrar

De la práctica, sin pretensión de exhaustividad: un conector de prueba de una migración que lleva años activo. Una aplicación especializada que envía directamente al servicio, aunque todos creen que pasa por el gateway. Un proveedor de boletines cuyo contrato ha expirado, pero que todavía puede realizar entregas. Y, con regularidad, un conector con condiciones demasiado abiertas que alguien creó una vez para resolver un problema urgente.

Ninguno de estos hallazgos es dramático por sí solo. Juntos dibujan la imagen de un entorno que ya nadie controla por completo, y ese es precisamente el riesgo real.

## Límites del método

Debe conocer tres limitaciones.

El seguimiento de mensajes mediante el cmdlet solo se remonta unos diez días. Para periodos más largos necesita la búsqueda histórica, que se ejecuta de forma asíncrona y cubre hasta 90 días. De lo contrario, se le escaparán los sistemas poco frecuentes que envían una vez al mes.

`FromIP` no significa lo mismo en todas partes. Para correo procedente de Internet, es la dirección del servidor que realiza la entrega. Para correo híbrido, es la dirección de su servidor de transporte local, no la del remitente original. Por tanto, el análisis le muestra la **última estación antes del servicio**, no el origen.

Y la asignación a un conector no es visible directamente en el tenant. Se deduce a partir de la dirección, el certificado y el dominio del remitente. Para una afirmación fiable sobre el uso de un conector concreto, el informe de conectores del Exchange Admin Center, en Informes y Flujo de correo, es la mejor fuente, porque agrega los datos del lado del servidor durante periodos más largos.

## Como comprobación recurrente

Este análisis es muy adecuado como rutina trimestral. Guarde el resultado y compárelo en la siguiente ejecución. Las nuevas direcciones de la lista son cambios documentados o algo que querrá conocer.

Si de todos modos revisa la configuración de correo de sus dominios: el [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check) muestra SPF, DKIM, DMARC y los demás estándares de correo para cualquier dominio directamente en el navegador, también para dominios secundarios y de marketing que, según la experiencia, suelen olvidarse en estos inventarios. Y para las propias consultas, el [Generador de comandos](https://rafaelpfister.ch/tools/command-builder) proporciona bloques listos para PowerShell y Unix shell.

Puede consultar cómo seguir investigando mensajes individuales sospechosos en [Analizar el flujo de correo de Exchange: Message Tracking, protocolos SMTP y conectores de recepción](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren).

## Fuentes

1.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): lista de campos, incluidos FromIP y ToIP, así como la lógica de paginación con StartingRecipientAddress.

2.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): seguimiento asíncrono de mensajes durante hasta 90 días para periodos más antiguos.

3.  [Get-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchange/get-inboundconnector): parámetros de los conectores entrantes, incluidos SenderIPAddresses y TreatMessagesAsInternal.

4.  [Configure mail flow using connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): interacción entre los tipos de conectores y cuándo se aplica cada uno.

5.  [RFC 1918: Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918): define los rangos de direcciones privadas que debe distinguir de las direcciones públicas en el análisis.
