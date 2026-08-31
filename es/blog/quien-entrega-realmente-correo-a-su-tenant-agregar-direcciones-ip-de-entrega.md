---
title: "¿Quién entrega realmente correo a su tenant? Agregar direcciones IP de entrega"
navTitle: "IP de entrega"
description: "Un único análisis muestra qué sistemas entregan realmente correo a su tenant: conectores olvidados, aplicaciones que envían directamente y proveedores de servicios que nadie ha documentado, incluidos los errores habituales de análisis relacionados con la lógica de paginación y la interpretación."
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
translationSourceHash: 9209720819061360cb72bfa01ab6261e6af80e547a398c25f6802edfbe49bb6c
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:06:15.492Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/quien-entrega-realmente-correo-a-su-tenant-agregar-direcciones-ip-de-entrega
---

# ¿Quién entrega realmente correo a su tenant? Agregar direcciones IP de entrega

Casi ningún entorno de correo sabe ya por completo quién le entrega mensajes. Con los años se acumulan conectores de migraciones, aplicaciones que envían directamente, proveedores de servicios cuyos contratos vencieron hace tiempo y configuraciones de prueba que nunca se retiraron. Mientras el correo fluya, nadie lo nota.

Un único análisis aporta claridad: agrupar todos los mensajes entrantes por su dirección IP de origen. Se realiza en dos minutos y la lista de resultados suele ser sorprendente. Este artículo muestra la consulta, explica cómo obtenerla **completa** y, sobre todo, cómo interpretar correctamente las cifras. Porque la interpretación es la parte más difícil.

## Por qué merece la pena

La lista responde a cuatro preguntas que, de otro modo, habría que aclarar laboriosamente una por una. ¿Qué sistemas envían realmente a su tenant? ¿Todo pasa por las rutas que ha documentado o hay una segunda entrada? ¿Se sigue utilizando un conector que quiere retirar? Y: ¿una aplicación envía directamente al servicio, sin pasar por su gateway y, por tanto, evitando su filtrado?

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

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-StartDate (Get-Date).AddHours(-2)` | Inicio de la ventana de consulta; aquí, hace dos horas |
| `-EndDate (Get-Date)` | Fin de la ventana de consulta; el momento actual |
| `-ResultSize 5000` | Número máximo de filas por llamada; 5000 es también el valor máximo |
| `Group-Object FromIP` | Agrupa los mensajes por la dirección IP que entrega el correo |
| `Sort-Object Count -Descending` | Ordena los grupos de forma descendente por número de mensajes |
| `Format-Table Count, Name -AutoSize` | Salida de dos columnas (cantidad, dirección IP) con ancho de columna automático |

</details>

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

## Fuente de error 1: la lista casi siempre está incompleta

`Get-MessageTraceV2` devuelve resultados paginados, con un máximo de 5000 filas por llamada. Con un volumen elevado, una página cubre solo una fracción de su ventana temporal. Entonces agrupa una muestra y considera el resultado como el total.

Puede reconocerlo por esta advertencia:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

**Si aparece esta advertencia, su análisis no sirve de nada.** En particular, una entrada ausente no debe interpretarse como ausencia. Una dirección con tres mensajes al día tampoco aparecerá en una muestra.

Hay dos soluciones. La sencilla: reduzca la ventana temporal hasta que deje de aparecer la advertencia. Con 5000 mensajes por hora, serán 55 minutos y no siete días. Para responder a la pregunta «¿qué sistemas envían realmente?», una ventana corta y completa suele bastar.

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

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-StartDate` / `-EndDate` | Ventana de consulta; aquí, las últimas 24 horas |
| `-StartingRecipientAddress` | Punto de continuación de la lógica de paginación: la dirección del destinatario a partir de la cual empieza la siguiente página |
| `-ResultSize 5000` | Tamaño de página; una página completa indica que hay más resultados |
| `Group-Object FromIP` | Agrupa el conjunto completo por la dirección IP que entrega el correo |
| `Sort-Object Count -Descending` | Ordena los grupos de forma descendente por número de mensajes |
| `Format-Table Count, Name -AutoSize` | Muestra la cantidad por dirección con ancho de columna automático |

</details>

El bucle recupera más páginas mientras una página devuelva exactamente 5000 filas y continúa cada vez con la última dirección de destinatario de la página anterior; solo entonces agrupa el conjunto completo.

Para 24 horas en un entorno mediano, calcule unos minutos de ejecución. Para un inventario puntual, es tiempo bien invertido.

## Fuente de error 2: las cifras no significan lo que parecen significar

La lista de resultados contiene cuatro tipos de entradas completamente distintos, y quien los mezcle sacará conclusiones erróneas.

**`255.255.255.255` no representa un sistema.** Este valor aparece cuando no hubo una conexión SMTP entrante desde el exterior para el mensaje. Afecta a los mensajes generados por el propio servicio: informes de diario, notificaciones de no entrega, respuestas automáticas de ausencia y mensajes entre buzones del mismo tenant. En casi todos los entornos es la entrada más grande y es completamente normal.

**Las direcciones privadas de RFC 1918** proceden de su propia red. En entornos híbridos verá aquí los servidores de transporte locales, ya que su dirección interna se conserva al transferir al servicio. Son las cifras grandes de la lista y, por regla general, la ruta principal esperada.

**Las direcciones de servicios de seguridad y filtrado** se reconocen por el operador, no por su valor numérico. Los proxies en la nube, los gateways de correo anteriores y los servicios de seguridad web aparecen con muchas direcciones contiguas y cifras medias. Normalmente forman parte de la configuración, pero deberían figurar en el manual de operaciones.

**Las direcciones públicas individuales con cifras bajas** son las interesantes. Precisamente ahí se esconden las aplicaciones olvidadas, los proveedores antiguos y los sistemas de los que ya nadie se acuerda.

## La resolución: convertir direcciones en nombres

Para todo lo que no pueda asignar de inmediato, resulta útil la resolución inversa. No siempre está configurada ni es siempre fiable, pero en la mayoría de los casos proporciona la pista decisiva:

```powershell
$unbekannt = '198.51.100.77','203.0.113.9'

$unbekannt | ForEach-Object {
    $name = try { (Resolve-DnsName $_ -Type PTR -ErrorAction Stop).NameHost } catch { '(kein PTR)' }
    [pscustomobject]@{ IP = $_; Name = $name }
} | Format-Table -AutoSize
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Resolve-DnsName $_ -Type PTR` | Consulta el registro inverso (PTR) de cada dirección IP |
| `-ErrorAction Stop` | Convierte una entrada inexistente en un error capturable para el bloque `try`/`catch` |
| `[pscustomobject]@{ … }` | Crea un objeto por dirección con la IP y el nombre resuelto para la salida de tabla |
| `Format-Table -AutoSize` | Salida con ancho de columna automático |

</details>

```text
IP            Name
--            ----
198.51.100.77 mail-out-03.newsletter-provider.example
203.0.113.9   (kein PTR)
```

La ausencia de un PTR no es en sí misma indicio de un problema, pero sí es una buena razón para examinarlo más de cerca. Para esas direcciones, revise los mensajes correspondientes:

```powershell
Get-MessageTraceV2 -StartDate (Get-Date).AddHours(-2) -EndDate (Get-Date) -ResultSize 5000 |
  Where-Object { $_.FromIP -eq '203.0.113.9' } |
  Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-StartDate` / `-EndDate` / `-ResultSize` | Ventana de consulta y tamaño de página como en la consulta principal |
| `Where-Object { $_.FromIP -eq '203.0.113.9' }` | Filtra en el cliente por la dirección de origen en cuestión |
| `Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize` | Muestra por mensaje la hora de recepción, el remitente, el destinatario, el asunto y el estado de entrega |

</details>

Por regla general, el remitente y el asunto le indicarán inmediatamente qué aplicación hay detrás.

## La comparación: ¿qué dirección corresponde a qué conector?

Compare su lista de resultados con los conectores configurados:

```powershell
Get-InboundConnector |
    Format-List Name, Enabled, ConnectorType, ConnectorSource,
        @{n='SenderIPAddresses'; e={$_.SenderIPAddresses -join ', '}},
        @{n='SenderDomains';     e={$_.SenderDomains -join ', '}},
        RequireTls, TlsSenderCertificateName,
        RestrictDomainsToCertificate, CloudServicesMailEnabled, TreatMessagesAsInternal
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Get-InboundConnector` | Enumera todos los conectores entrantes del tenant; deliberadamente sin parámetros restrictivos |
| `Format-List <Eigenschaften>` | Salida como lista de las propiedades indicadas, una por línea |
| `@{n='…'; e={…}}` | Propiedad calculada con nombre (`n`) y expresión (`e`) |
| `-join ', '` | Convierte la matriz de direcciones o dominios en una línea legible separada por comas |

</details>

Tres situaciones son reveladoras.

**Una dirección entrega correo, pero no figura en ningún conector.** Entonces el correo entra como correo de Internet normal. Esto está permitido, pero significa que esa aplicación no recibe ningún trato especial y que sus mensajes están sujetos al filtrado completo. Si alguien afirma que hay un conector configurado para este sistema, claramente ya no es así.

**Un conector menciona direcciones de las que no llega nada.** Es candidato a retirarse. Antes de eliminarlo, compruebe si se trata de sistemas estacionales o poco frecuentes y desactívelo primero, en lugar de quitarlo de inmediato.

**Un conector establece `TreatMessagesAsInternal` o `CloudServicesMailEnabled` en verdadero.** Aquí conviene examinarlo detenidamente. Ambas configuraciones hacen que los mensajes que llegan por esta ruta se traten como internos de la organización. Si por ella entra correo desde Internet, se omiten las comprobaciones pensadas para mensajes externos, entre ellas la protección contra remitentes falsificados de su propio dominio. Para un conector híbrido puro, esto es correcto; para un conector mediante el cual entregan sistemas arbitrarios, es un hallazgo.

## Lo que suele encontrar

De la práctica, sin pretensión de exhaustividad: un conector de prueba de una migración que lleva años activo. Una aplicación especializada que envía directamente al servicio, aunque todos creen que pasa por el gateway. Un proveedor de boletines cuyo contrato expiró, pero que sigue pudiendo entregar correo. Y, con regularidad, un conector con condiciones demasiado abiertas que alguien creó en su momento para resolver un problema urgente.

Ninguno de estos hallazgos es dramático por sí solo. Juntos dibujan la imagen de un entorno que ya nadie controla por completo, y ese es precisamente el verdadero riesgo.

## Límites del método

Debe conocer tres limitaciones.

El seguimiento de mensajes mediante el cmdlet solo llega unos diez días atrás. Para periodos más largos necesita la búsqueda histórica, que se ejecuta de forma asíncrona y cubre hasta 90 días. De lo contrario, se le escaparán los sistemas poco frecuentes que envían mensualmente.

`FromIP` no significa lo mismo en todas partes. En el correo de Internet, es la dirección del servidor que entrega el mensaje. En el correo híbrido, es la dirección de su servidor de transporte local, no la del remitente original. Por tanto, el análisis le muestra la **última estación antes del servicio**, no el origen.

Y la asignación a un conector no es visible directamente en el tenant. Se deduce a través de la dirección, el certificado y el dominio del remitente. Para una afirmación fiable sobre el uso de un conector individual, el informe de conectores del Centro de administración de Exchange, en Informes y Flujo de correo, es la mejor fuente, porque agrega los datos del lado del servidor durante periodos más largos.

## Como comprobación recurrente

Este análisis resulta adecuado como rutina trimestral. Guarde el resultado y compárelo en la siguiente ejecución. Las direcciones nuevas en la lista son cambios documentados o algo que querrá conocer.

Si de todos modos revisa la configuración de correo de sus dominios: el [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check) muestra SPF, DKIM, DMARC y los demás estándares de correo para cualquier dominio directamente en el navegador, también para dominios secundarios y de marketing que, según la experiencia, suelen olvidarse en estos inventarios. Y para las propias consultas, el [Generador de comandos](https://rafaelpfister.ch/tools/command-builder) proporciona bloques listos para PowerShell y shell Unix.

Cómo seguir investigando mensajes individuales sospechosos se explica en [Analizar el flujo de correo de Exchange: Message Tracking, registros SMTP y conectores de recepción](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren).

## Fuentes

1.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): lista de campos, incluidos FromIP y ToIP, así como la lógica de paginación con StartingRecipientAddress.

2.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): seguimiento asíncrono de mensajes durante hasta 90 días para periodos anteriores.

3.  [Get-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchange/get-inboundconnector): parámetros de los conectores entrantes, incluidos SenderIPAddresses y TreatMessagesAsInternal.

4.  [Configurar el flujo de correo mediante conectores en Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): interacción entre los tipos de conector y cuándo se aplica cada uno.

5.  [RFC 1918: asignación de direcciones para Internets privados](https://www.rfc-editor.org/rfc/rfc1918): define los rangos de direcciones privadas que debe distinguir de las direcciones públicas en el análisis.
