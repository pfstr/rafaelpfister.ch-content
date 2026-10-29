---
title: "¿Quién entrega realmente correo a su tenant? Agregar direcciones IP de entrega"
navTitle: "IPs de entrega"
description: "Un único análisis muestra qué sistemas entregan realmente correo a su tenant: conectores olvidados, aplicaciones que envían directamente y proveedores que nadie ha documentado, incluidos los errores habituales de análisis relacionados con la lógica de paginación y la interpretación."
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
translationSourceHash: 291ed92e756d5fa011212a5cf9d3ce9061df96d73857f508c838183965ba25b4
translationModel: gpt-5.6-terra
translatedAt: 2026-10-29T03:39:12.991Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/quien-entrega-realmente-correo-a-su-tenant-agregar-direcciones-ip-de-entrega
---

# ¿Quién entrega realmente correo a su tenant? Agregar direcciones IP de entrega

Casi ningún entorno de correo sabe ya por completo quién le entrega correo. Con los años se acumulan conectores de migraciones, aplicaciones que envían directamente, proveedores cuyos contratos expiraron hace tiempo y configuraciones de prueba que nunca se desmontaron. Mientras el correo fluya, nadie lo nota.

Un único análisis aporta claridad: la agrupación de todos los mensajes entrantes por su dirección IP de origen. Se realiza en dos minutos y la lista de resultados suele ser sorprendente. Este artículo muestra la consulta, explica cómo obtenerla **completa** y, sobre todo, cómo interpretar correctamente las cifras. Porque la interpretación es la parte más difícil.

## Por qué vale la pena

La lista responde a cuatro preguntas que, de otro modo, cuesta aclarar por separado. ¿Qué sistemas envían realmente a su tenant? ¿Todo pasa por las vías que ha documentado o existe una segunda entrada? ¿Sigue utilizándose un conector que desea retirar? Y: ¿una aplicación envía directamente al servicio, saltándose su gateway y, por tanto, su filtrado?

La última pregunta, en particular, es relevante para la seguridad. Quien entrega directamente no solo evita el filtrado, sino a menudo también el registro en el que desea poder confiar en caso de incidente.

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
| `-StartDate (Get-Date).AddHours(-2)` | Inicio de la ventana de consulta, aquí hace dos horas |
| `-EndDate (Get-Date)` | Fin de la ventana de consulta, el momento actual |
| `-ResultSize 5000` | número máximo de filas por llamada; 5000 es también el valor máximo |
| `Group-Object FromIP` | agrupa los mensajes por la dirección IP de entrega |
| `Sort-Object Count -Descending` | ordena los grupos de forma descendente por número de mensajes |
| `Format-Table Count, Name -AutoSize` | salida en dos columnas (cantidad, dirección IP) con ancho de columna automático |

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

Antes de extraer conclusiones, deben cumplirse dos condiciones: la lista debe estar completa y debe saber qué significan las entradas.

## Fuente de error 1: la lista casi siempre está incompleta

`Get-MessageTraceV2` devuelve resultados por páginas, con un máximo de 5000 filas por llamada. Con un volumen elevado, una página cubre solo una fracción de su ventana temporal. Entonces agrupa un subconjunto y toma el resultado por el total.

Puede reconocerlo por esta advertencia:

```text
WARNING: There are more results, use the following command to get more.
Get-MessageTraceV2 -StartDate "2026-08-11T07:25:19Z" -EndDate "2026-08-11T09:05:46Z"
  -StartingRecipientAddress "naechster@example.com" -ResultSize 5000
```

**Si aparece esta advertencia, su análisis no sirve para nada.** En particular, una entrada ausente no debe interpretarse como ausencia. Una dirección con tres mensajes al día tampoco aparecerá en un subconjunto.

Hay dos salidas. La sencilla: reduzca la ventana temporal hasta que deje de aparecer la advertencia. Con 5000 mensajes por hora, son 55 minutos y no siete días. Para la pregunta «¿qué sistemas envían realmente?», una ventana corta pero completa suele bastar perfectamente.

La vía exhaustiva recorre todas las páginas y recopila los resultados:

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
    if ($seite.Count -eq 5000) {
        $naechster = $seite[-1].RecipientAddress
        $ende      = $seite[-1].Received
    } else {
        $naechster = $null
    }
    Write-Host "Gesammelt: $($alle.Count)"
} while ($naechster)

$alle | Group-Object FromIP | Sort-Object Count -Descending |
    Format-Table Count, Name -AutoSize
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-StartDate` / `-EndDate` | Ventana de consulta, aquí las últimas 24 horas |
| `-StartingRecipientAddress` | Punto de continuación de la lógica de paginación: la dirección de destinatario de la última entrada de la página anterior; junto con `-EndDate` = su hora de recepción (`Received`) comienza allí la página siguiente |
| `-ResultSize 5000` | tamaño de página; una página completa indica que siguen más resultados |
| `Group-Object FromIP` | agrupa el conjunto completo por la dirección IP de entrega |
| `Sort-Object Count -Descending` | ordena los grupos de forma descendente por número de mensajes |
| `Format-Table Count, Name -AutoSize` | salida de la cantidad por dirección con ancho de columna automático |

</details>

El bucle recupera más páginas mientras una página devuelva exactamente 5000 filas, y cada vez continúa con la dirección de destinatario y la hora de recepción de la última entrada de la página anterior, tal como prevé la documentación de `Get-MessageTraceV2`; solo entonces se agrupa el conjunto completo.

Para 24 horas en un entorno mediano, calcule unos minutos de ejecución. Para un inventario puntual, es tiempo bien invertido.

## Fuente de error 2: las cifras no significan lo que parecen

La lista de resultados contiene cuatro tipos de entradas completamente distintos, y quien los mezcle extraerá conclusiones erróneas.

**`255.255.255.255` no representa un sistema.** Este valor aparece cuando no hubo una conexión SMTP entrante desde fuera para el mensaje. Afecta a mensajes generados por el propio servicio: informes de diario, notificaciones de no entrega, respuestas automáticas por ausencia y mensajes entre buzones del mismo tenant. En casi todos los entornos es la entrada más grande y es totalmente normal.

**Las direcciones privadas de RFC 1918** proceden de su propia red. En entornos híbridos verá aquí los servidores de transporte locales, pues su dirección interna se conserva al realizar la entrega al servicio. Son las cifras grandes de la lista y, por regla general, la vía principal esperada.

**Las direcciones de servicios de seguridad y filtrado** se reconocen por el operador, no por el valor numérico. Los proxies en la nube, los gateways de correo previos y los servicios de seguridad web aparecen con muchas direcciones contiguas y cifras intermedias. Normalmente forman parte de la configuración, pero deberían figurar en el manual operativo.

**Las direcciones públicas individuales con cifras pequeñas** son las interesantes. Ahí se esconden precisamente las aplicaciones olvidadas, los antiguos proveedores y los sistemas de los que ya nadie se acuerda.

## La resolución: de direcciones a nombres

Para todo lo que no pueda asignar de inmediato, ayuda la resolución inversa. No siempre está configurada ni es siempre fiable, pero en la mayoría de los casos proporciona la pista decisiva:

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
| `Resolve-DnsName $_ -Type PTR` | consulta el registro inverso (PTR) de cada dirección IP |
| `-ErrorAction Stop` | convierte una entrada ausente en un error que el bloque `try`/`catch` puede capturar |
| `[pscustomobject]@{ … }` | crea por dirección un objeto con IP y nombre resuelto para la salida en tabla |
| `Format-Table -AutoSize` | salida con ancho de columna automático |

</details>

```text
IP            Name
--            ----
198.51.100.77 mail-out-03.newsletter-provider.example
203.0.113.9   (kein PTR)
```

La ausencia de un PTR no es por sí sola indicio de un problema, pero sí un buen motivo para analizarlo más de cerca. Para esas direcciones, examine los mensajes correspondientes:

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
| `Where-Object { $_.FromIP -eq '203.0.113.9' }` | filtra en el cliente por la dirección de origen en cuestión |
| `Format-Table Received, SenderAddress, RecipientAddress, Subject, Status -AutoSize` | muestra por mensaje la hora de recepción, remitente, destinatario, asunto y estado de entrega |

</details>

Por regla general, el remitente y el asunto le indicarán enseguida qué aplicación hay detrás.

## La comparación: ¿qué dirección pertenece a qué conector?

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
| `Get-InboundConnector` | enumera todos los conectores entrantes del tenant; aquí deliberadamente sin parámetros restrictivos |
| `Format-List <Eigenschaften>` | salida como lista de las propiedades indicadas, una por línea |
| `@{n='…'; e={…}}` | propiedad calculada con nombre (`n`) y expresión (`e`) |
| `-join ', '` | convierte el array de direcciones o dominios en una línea legible separada por comas |

</details>

Tres situaciones son reveladoras.

**Una dirección entrega correo, pero no figura en ningún conector.** Entonces el correo llega como correo de Internet normal. Es válido, pero significa que esa aplicación no recibe ningún tratamiento especial y que sus mensajes están sujetos a todo el filtrado. Si alguien afirma que hay un conector configurado para ese sistema, evidentemente ya no es así.

**Un conector indica direcciones de las que no llega nada.** Es candidato para retirarlo. Antes de eliminarlo, compruebe si se trata de sistemas estacionales o poco frecuentes, y desactívelo primero en lugar de eliminarlo directamente.

**Un conector establece `TreatMessagesAsInternal` o `CloudServicesMailEnabled` en verdadero.** Aquí conviene examinarlo detenidamente. Ambas configuraciones hacen que los mensajes recibidos por esta vía se traten como internos de la organización. Si por ella entra correo de Internet, se eluden comprobaciones pensadas para mensajes externos, entre ellas la protección contra remitentes falsificados de su propio dominio. Para un conector híbrido puro es correcto; para un conector por el que entregan sistemas arbitrarios, es un hallazgo.

## Lo que suele encontrar

De la práctica, sin pretensión de exhaustividad: un conector de prueba de una migración que lleva años activo. Una aplicación de negocio que envía directamente al servicio, aunque todos creen que pasa por el gateway. Un proveedor de newsletters cuyo contrato expiró, pero que sigue pudiendo entregar correo. Y, con frecuencia, un conector con condiciones muy abiertas que alguien creó una vez para resolver un problema urgente.

Ninguno de estos hallazgos es dramático por sí solo. Juntos dibujan la imagen de un entorno que ya nadie controla por completo, y ese es precisamente el riesgo real.

## Límites del método

Debe conocer tres limitaciones.

`Get-MessageTraceV2` cubre los últimos 90 días, pero devuelve como máximo diez días por consulta. Para períodos más largos, consulte varias ventanas temporales o utilice la búsqueda histórica, que se ejecuta de forma asíncrona y también cubre hasta 90 días. De lo contrario, se le escaparán los sistemas poco frecuentes que envían mensualmente.

`FromIP` no significa lo mismo en todas partes. Para correo de Internet, es la dirección del servidor que realiza la entrega. Para correo híbrido, es la dirección de su servidor de transporte local, no la del remitente original. Por tanto, el análisis le muestra la **última estación antes del servicio**, no el origen.

Y la asignación a un conector no es visible directamente en el tenant. Se deduce a partir de la dirección, el certificado y el dominio del remitente. Para una afirmación fiable sobre el uso de un conector concreto, el informe de conectores del Centro de administración de Exchange, en Informes y flujo de correo, es la mejor fuente, ya que agrega del lado del servidor durante períodos más largos.

## Como comprobación periódica

Este análisis resulta adecuado como rutina trimestral. Guarde el resultado y compárelo en la siguiente ejecución. Las direcciones nuevas de la lista son cambios documentados o algo que querrá conocer.

Si de todos modos revisa la configuración de correo de sus dominios: el [Mail-DNS-Check](https://rafaelpfister.ch/tools/mail-check) muestra SPF, DKIM, DMARC y los demás estándares de correo para cualquier dominio directamente en el navegador, también para dominios secundarios y de marketing que, por experiencia, se olvidan en este tipo de inventarios. Y para las propias consultas, el [Generador de comandos](https://rafaelpfister.ch/tools/command-builder) ofrece bloques listos para usar en PowerShell y shell Unix.

En [Analizar el flujo de correo de Exchange: seguimiento de mensajes, protocolos SMTP y conectores de recepción](https://rafaelpfister.ch/blog/exchange-message-tracking-und-receive-connectoren-analysieren) se explica cómo seguir investigando mensajes individuales llamativos.

## Historial de cambios

- **25 de octubre de 2026:** Corregido: la lógica de paginación de Get-MessageTraceV2 continúa con la dirección de destinatario y la hora de recepción (EndDate) de la última entrada. Corregido: el cmdlet cubre los últimos 90 días, pero devuelve como máximo diez días por consulta.

## Fuentes

1.  [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchange/get-messagetracev2): lista de campos, incluidos FromIP y ToIP, período de consulta (90 días, como máximo diez días por consulta) y continuación con StartingRecipientAddress y EndDate.

2.  [Start-HistoricalSearch](https://learn.microsoft.com/en-us/powershell/module/exchange/start-historicalsearch): seguimiento asíncrono de mensajes durante hasta 90 días para períodos más antiguos.

3.  [Get-InboundConnector](https://learn.microsoft.com/en-us/powershell/module/exchange/get-inboundconnector): parámetros de los conectores entrantes, entre ellos SenderIPAddresses y TreatMessagesAsInternal.

4.  [Configure mail flow using connectors in Exchange Online](https://learn.microsoft.com/en-us/exchange/mail-flow-best-practices/use-connectors-to-configure-mail-flow/use-connectors-to-configure-mail-flow): interacción entre los tipos de conectores y cuándo se aplica cada uno.

5.  [RFC 1918: Address Allocation for Private Internets](https://www.rfc-editor.org/rfc/rfc1918): define los rangos de direcciones privadas que debe distinguir de las direcciones públicas en el análisis.
