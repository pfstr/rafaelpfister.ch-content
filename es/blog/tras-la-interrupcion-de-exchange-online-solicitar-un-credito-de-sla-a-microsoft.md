---
title: "Tras la caída de Exchange Online: solicitar un crédito SLA a Microsoft"
navTitle: "Crédito SLA"
description: "La caída de Exchange Online del 12 y 13 de noviembre de 2026 duró unas 22 horas y, por tanto, sitúa a muchos tenants por debajo de la disponibilidad garantizada del 99,9 %. Microsoft no abona el crédito automáticamente: debe solicitarse antes del 31 de diciembre de 2026. Cómo calcular la disponibilidad, qué pruebas exige Microsoft y por qué las pequeñas empresas deben hacerlo a través de su partner y las grandes directamente con Microsoft."
date: "2026-11-18"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min de lectura"
themen:
  - microsoft-365-exchange
produkte:
  - "exchange-online"
protokolle:
  - "lizenzierung"
  - "powershell"
slug: "tras-la-interrupcion-de-exchange-online-solicitar-un-credito-de-sla-a-microsoft"
translationId: "article-0e96d35615b0d8d9"
translationOf: microsoft-sla-gutschrift-einfordern
translationSourceHash: e2a540fa8b0a15956e2a9f4af6b604f517d4f077ec2d566bd2dd187a1d423216
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T11:22:20.021Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/tras-la-interrupcion-de-exchange-online-solicitar-un-credito-de-sla-a-microsoft
---

# Tras la caída de Exchange Online: solicitar un crédito SLA a Microsoft

<div class="notfall-hinweis notfall-hinweis--gross">
<p class="notfall-hinweis__titel">Asistencia gratuita para la solicitud de SLA</p>
<p>Le ayudamos gratuitamente a solicitar a Microsoft o a su partner el crédito por la caída de Exchange Online del 12 y 13 de noviembre de 2026: calculamos la disponibilidad, recopilamos las pruebas y presentamos la solicitud. El plazo vence el 31 de diciembre de 2026. Póngase en contacto mediante el <a href="https://rafaelpfister.ch/kontakt">formulario de contacto</a>.</p>
</div>

Del 12 de noviembre de 2026 a las 18:54 hasta el 13 de noviembre de 2026 a las 17:10 (hora suiza), muchas organizaciones no tuvieron acceso o solo tuvieron acceso intermitente a sus buzones de Exchange Online. Microsoft registró la incidencia en el Centro de administración bajo EX1550543 (Exchange Online) y MO1550821 (otros servicios de Microsoft 365). Documenté el desarrollo en el [seguimiento en directo de la caída](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11). Para Exchange Online, Microsoft garantiza una disponibilidad mensual del 99,9 %. Una caída de más de 22 horas está claramente por debajo de ese nivel. Sin embargo, solo recibirá un crédito si lo solicita, y el procedimiento depende de cómo haya adquirido su organización Microsoft 365.

## Lo que garantiza el SLA

El documento determinante es el «Service Level Agreement for Microsoft Online Services», que Microsoft publica de nuevo cada mes. Este artículo se basa en la versión del 17 de noviembre de 2026. Para Exchange Online, el SLA define un tiempo de inactividad como cualquier período en el que los usuarios no puedan enviar ni recibir correos electrónicos con Outlook en la web. Exchange Online no dispone de ventanas de mantenimiento programadas.

La disponibilidad se calcula en minutos de usuario:

```text
Verfügbarkeit (%) = (Benutzerminuten - Ausfallminuten) / Benutzerminuten x 100
```

Los minutos de usuario son los minutos del mes multiplicados por el número de usuarios. Los minutos de inactividad son la duración de cada incidente multiplicada por el número de usuarios afectados. Si la disponibilidad cae por debajo de un umbral, se concede un crédito sobre la cuota mensual del servicio afectado:

| Disponibilidad mensual | Crédito |
|---|---|
| menos del 99,9 % | 25 % |
| menos del 99 % | 50 % |
| menos del 95 % | 100 % |

Para Microsoft Teams, SharePoint Online y OneDrive se aplican los mismos umbrales, con sus propias definiciones de tiempo de inactividad. En Teams, solo cuenta si los usuarios no pueden chatear ni iniciar reuniones. Según Microsoft, durante EX1550543 los chats y las reuniones funcionaron en gran medida; se vieron afectados los calendarios y las invitaciones. Por ello, una solicitud para Teams probablemente tenga pocas posibilidades.

Además de la disponibilidad del buzón, Exchange Online cuenta con otros niveles de servicio, como el tiempo de entrega dentro de Microsoft 365 y la aceptación de correos entrantes. Si un incidente afecta a varios de ellos, según el SLA deberá elegir uno. Para la caída de noviembre, la disponibilidad del buzón a través de Outlook en la web es el criterio más lógico.

## Qué supone la caída en porcentajes

Noviembre de 2026 tiene 30 días, es decir, 43'200 minutos por usuario. De ello se derivan estos valores límite por usuario:

| Umbral | Tiempo de inactividad permitido en noviembre |
|---|---|
| 99,9 % | 43 minutos |
| 99 % | 7 horas y 12 minutos |
| 95 % | 36 horas |

La incidencia duró 1'336 minutos desde el primer aviso hasta su resolución. Dos ejemplos de cálculo para una organización con 200 buzones:

| Caso | Minutos de inactividad | Disponibilidad | Crédito |
|---|---|---|---|
| Los 200 usuarios sin acceso durante toda la incidencia | 200 x 1'336 = 267'200 | 96,91 % | 50 % |
| 120 usuarios sin acceso durante 10 horas | 120 x 600 = 72'000 | 99,17 % | 25 % |

El número total de minutos de usuario es, en ambos casos, 200 x 43'200 = 8'640'000. Lo decisivo es cuántos usuarios estuvieron realmente afectados y durante cuánto tiempo. Microsoft lo verifica con su propia telemetría, pero en la solicitud exige que facilite sus datos al respecto. Muchos tenants en Europa se vieron afectados solo a partir de la noche del 12 de noviembre o de forma intermitente. Por tanto, calcule con los períodos que pueda demostrar, no con la duración total del aviso de Microsoft.

## Qué es el crédito y qué no es

El SLA regula varios aspectos que conviene tener claros antes de presentar la solicitud:

1.  **Solo previa solicitud.** Microsoft no revisa ni concede créditos por iniciativa propia. Según el SLA, un aviso de estado de Microsoft no constituye expresamente una confirmación de que no se haya cumplido un nivel de servicio.

2.  **Crédito, no reembolso.** El crédito se compensará con una factura futura. Está limitado a la cuota mensual del servicio afectado.

3.  **Único recurso.** Según el SLA, los créditos son el único recurso para los problemas de disponibilidad. Microsoft no compensa pérdidas de ingresos, horas extra ni otros costes derivados. No puede reducir las facturas por su cuenta.

4.  **Solo el servicio afectado.** Si dispone de una suite como Microsoft 365 E3 o Business Premium, el crédito se refiere a la parte correspondiente a Exchange Online, no al precio total de la licencia. Microsoft determina esa proporción durante la revisión; el SLA no especifica ninguna fórmula.

5.  **Sin crédito para ofertas gratuitas y de prueba.** Quedan excluidas las versiones de prueba, las versiones preliminares y los servicios gratuitos, así como las licencias donadas a organizaciones sin ánimo de lucro. Los clientes particulares con Microsoft 365 Personal o Family no están cubiertos por este SLA.

6.  **Contratos Open en días.** Quienes tengan licencias mediante Open, Open Value u Open Value Subscription recibirán cualquier crédito como días de uso adicionales en lugar de un importe monetario.

7.  **Plazo.** Para los servicios de Microsoft 365, la solicitud con todos los datos debe recibirse antes de que termine el mes posterior al mes del incidente. Para la caída de noviembre, la fecha límite es el **31 de diciembre de 2026**. Para Azure se aplica un plazo propio de 60 días.

## Qué datos exige Microsoft

El SLA establece cinco datos obligatorios. Si faltan, Microsoft rechazará la solicitud:

1.  una descripción precisa del incidente,
2.  el momento y la duración del tiempo de inactividad,
3.  los recursos afectados (en Exchange Online: tenant y servicio),
4.  el número y las ubicaciones de los usuarios afectados,
5.  una descripción de los errores producidos.

El identificador del incidente (EX1550543) y el ID del tenant deben incluirse siempre. Ambos pueden consultarse con Microsoft Graph PowerShell:

```powershell
Connect-MgGraph -Scopes "ServiceHealth.Read.All", "Organization.Read.All"
(Get-MgOrganization).Id
Get-MgServiceAnnouncementIssue -ServiceHealthIssueId "EX1550543" |
    Select-Object Id, Title, StartDateTime, EndDateTime, Classification
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Scopes "ServiceHealth.Read.All"` | Permiso para leer los avisos de estado de los servicios |
| `-Scopes "Organization.Read.All"` | Permiso para leer los datos de la organización, incluido el ID del tenant |
| `(Get-MgOrganization).Id` | muestra el ID del tenant |
| `-ServiceHealthIssueId "EX1550543"` | recupera exactamente este incidente |
| `Select-Object …` | muestra el identificador, el título, el inicio, el final y la clasificación del incidente |

</details>

La clasificación (`Classification`) indica si Microsoft registra el incidente como `incident` o como `advisory`. Según la documentación para partners de Microsoft, los avisos de nivel «Advisory» no suelen dar derecho a crédito. EX1550543 se clasificó como Incident.

Para el número y la ubicación de los usuarios, basta con una evaluación de los buzones con licencia por ubicación de uso:

```powershell
Connect-MgGraph -Scopes "User.Read.All"
$abfrage = @{
    All              = $true
    Filter           = 'assignedLicenses/$count ne 0'
    ConsistencyLevel = 'eventual'
    CountVariable    = 'anzahl'
    Property         = 'UsageLocation'
}
Get-MgUser @abfrage |
    Group-Object UsageLocation |
    Select-Object Name, Count
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `$abfrage = @{ … }` | recopila los parámetros para `Get-MgUser` en una tabla hash (splatting), para mantener la línea corta |
| `All` | lee todos los usuarios, no solo la primera página |
| `Filter 'assignedLicenses/$count ne 0'` | solo usuarios con al menos una licencia asignada |
| `ConsistencyLevel eventual` | necesario para el filtro `$count` (consulta avanzada) |
| `CountVariable anzahl` | guarda el número total en la variable `$anzahl` |
| `Property UsageLocation` | carga también la ubicación de uso |
| `Group-Object UsageLocation` | cuenta los usuarios por país |

</details>

Esta cifra es el límite máximo. Demuestre cuántos usuarios se quedaron realmente sin acceso con sus propios datos: tickets de helpdesk con marca de tiempo, mensajes de error de Outlook y Outlook en la web (capturas de pantalla), resultados de pruebas de disponibilidad de su monitorización y registros de inicio de sesión y acceso. Cuanto más precisos sean estos datos, menos dependerá el resultado únicamente de la telemetría de Microsoft.

## Pequeñas empresas: normalmente a través del partner

Las pequeñas y medianas empresas suelen adquirir Microsoft 365 en Suiza a través de un partner del Cloud Solution Provider Program (CSP) o directamente con tarjeta de crédito en el Centro de administración. Puede comprobar qué vía se aplica en el Centro de administración de Microsoft 365, en **Facturación > Sus productos**: si aparece un partner como fuente de compra, el contrato es un contrato CSP.

### Comprado a través de un partner CSP

En este caso, Microsoft no acepta la solicitud directamente de usted. Según Partner Center, solo los proveedores directos de CSP y los proveedores indirectos (distribuidores) pueden solicitar créditos; los revendedores indirectos no. Por tanto, si su proveedor de servicios de TI es un revendedor indirecto, la solicitud pasa a través de él a su distribuidor y de ahí a Microsoft. Cada nivel tiene sus propios plazos, a menudo anteriores. Por ello, contacte ahora y no espere hasta finales de diciembre.

Su partner necesita de usted:

1.  el ID del tenant,
2.  el identificador del incidente (EX1550543),
3.  el período, el número y las ubicaciones de los usuarios afectados, así como los errores observados,
4.  un correo electrónico en el que solicite el crédito SLA. Debe enviarse desde una dirección del dominio afectado; Microsoft no acepta correos desde una dirección privada.

Microsoft ya conoce el incidente. La solicitud debe demostrar que su organización se vio afectada: según Microsoft, el nombre, el ID del tenant y el número de ticket por sí solos no bastan. Si se concede el crédito, Microsoft lo abonará al partner y este se lo transferirá a usted. Si lo hace y cómo lo hace se rige por su contrato con el partner. Pregunte si no aparece nada en la próxima factura.

### Comprado directamente a Microsoft

Si ha pagado Microsoft 365 directamente a Microsoft, presente usted mismo la solicitud en el Centro de administración. Para ello necesita un rol de administrador:

1.  En `admin.cloud.microsoft`, abra **Ayuda y soporte técnico** en la parte inferior derecha.
2.  Describa la solicitud, por ejemplo, «Request an SLA service credit for a Microsoft 365 outage», y seleccione **Contactar con soporte técnico**.
3.  Elija el correo electrónico como vía de contacto. El título debe incluir «SLA service credit request» y el identificador EX1550543.
4.  En la descripción, indique los cinco datos obligatorios de la sección anterior y adjunte las pruebas.

Se requiere una solicitud independiente para cada servicio e incidente. Según el SLA, Microsoft suele tramitar las solicitudes en un plazo de 45 días.

### ¿Merece la pena para las pequeñas empresas?

Con diez usuarios, el crédito es pequeño: en una suite solo cuenta la parte de Exchange, de la que se concede el 25 o el 50 % durante un mes. El esfuerzo también es reducido si recopila ahora los datos. La solicitud tiene una segunda utilidad: documenta ante el partner y la dirección hasta qué punto se vio afectada su organización. Esto ayuda posteriormente a decidir si es necesario un sistema alternativo para el correo electrónico.

## Grandes empresas: directamente con Microsoft y con datos de medición propios

Las grandes organizaciones suelen licenciar Microsoft 365 mediante un Enterprise Agreement (EA) o el Microsoft Customer Agreement for Enterprise (MCA-E). En este caso, la solicitud se dirige directamente a Microsoft, ya sea a través del Centro de administración como se ha descrito anteriormente o mediante el contrato de Unified Support. Si tiene un Customer Success Account Manager (CSAM), infórmele de la solicitud; podrá hacer seguimiento de su tramitación. Si tiene un EA a través de un Licensing Solution Partner (LSP), aclare también en qué factura aparecerá el crédito.

Con varios miles de usuarios, se trata de importes considerables, y Microsoft revisa los datos con el correspondiente nivel de detalle. La diferencia respecto a un tenant pequeño está en la demostración:

1.  **Determinar con precisión los usuarios afectados.** Los grandes tenants se distribuyen entre varias bases de datos y regiones de los centros de datos de Microsoft; no todos los buzones estuvieron afectados durante el mismo tiempo. Declarar de forma general que todos los usuarios estuvieron afectados es cuestionable en tenants grandes. Son más sólidos los datos de medición: pruebas sintéticas contra Outlook en la web por ubicación, datos de disponibilidad de la monitorización propia y análisis de los tickets de helpdesk por ubicación y hora.

2.  **Indicar las ubicaciones.** El SLA exige las ubicaciones de los usuarios afectados. En organizaciones internacionales, conviene desglosar por país, ya que la incidencia afectó primero a Norteamérica y Sudamérica, y solo después a otras regiones.

3.  **Delimitar correctamente los entornos híbridos.** El SLA excluye los retrasos en el propio centro de datos, por ejemplo, correos en la cola de servidores Exchange locales hacia el tenant. Demuestre que el error se produjo dentro de Microsoft 365. Las colas de los servidores locales son una buena prueba, porque muestran que Exchange Online no aceptó los correos o los aceptó con retraso.

4.  **Elegir el nivel de servicio.** Si su principal problema fueron los tiempos de entrega y no tanto el acceso al buzón, el nivel de servicio para el tiempo de entrega puede ser una base mejor. Solo se permite una solicitud por incidente y servicio, por lo que debe elegir antes de presentarla.

5.  **Involucrar a compras y gestión de licencias.** El crédito aparece en la factura, no en el centro de costes de TI. Coordine con compras quién presenta la solicitud, quién comprueba el crédito y cómo se contabiliza.

Las grandes organizaciones también deberían prepararse para la próxima caída: una plantilla con los cinco datos obligatorios, una monitorización que mida la disponibilidad de Outlook en la web por ubicación y una responsabilidad clara para las solicitudes SLA.

## Plantilla para la solicitud

La siguiente plantilla contiene los datos obligatorios del SLA. Microsoft tramita más rápido las solicitudes en inglés:

```text
Subject: SLA service credit request: EX1550543, Exchange Online

Tenant ID: <Tenant-ID>
Service: Exchange Online
Incident ID: EX1550543 (related: MO1550821)
Service level: Exchange Online availability (Outlook on the web)

Description of the incident:
Users were unable to send or receive email using Outlook on the web,
Outlook for Windows and mobile clients.

Time and duration of downtime:
From 2026-11-12 <hh:mm> UTC to 2026-11-13 <hh:mm> UTC (<n> minutes).

Affected users:
<n> of <total> licensed users, locations: <Land: Anzahl, ...>

Errors observed:
<Fehlermeldungen, z. B. Verbindungsfehler in Outlook, HTTP 5xx in Outlook on the web>

Uptime calculation (November 2026):
(<total> x 43,200 - <Ausfallminuten>) / (<total> x 43,200) x 100 = <xx.xx> %

Attachments: helpdesk ticket export, monitoring data, screenshots.
```

## Fuentes

1.  [Service Level Agreement for Microsoft Online Services (resumen de versiones)](https://www.microsoft.com/licensing/docs/view/Service-Level-Agreements-SLA-for-Online-Services): versión del 17 de noviembre de 2026, definición del tiempo de inactividad para Exchange Online, niveles de crédito, datos obligatorios, plazos y exclusiones.

2.  [Microsoft Learn: How and when to request a credit from Microsoft](https://learn.microsoft.com/en-us/partner-center/billing/request-credit): procedimiento de solicitud para partners CSP, exclusión de revendedores indirectos, pruebas requeridas, clasificación de Advisories.

3.  [Microsoft Learn: Get-MgServiceAnnouncementIssue](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.devices.serviceannouncement/get-mgserviceannouncementissue): cmdlet para recuperar un incidente de estado del servicio mediante Microsoft Graph.

4.  [Microsoft 365 Status (status.cloud.microsoft)](https://status.cloud.microsoft/): avisos públicos de estado sobre EX1550543 y MO1550821.

5.  [Windows Latest: Microsoft 365 outage, you may get money off your bill](https://www.windowslatest.com/2026/11/13/microsoft-365-outage-you-may-get-money-off-your-bill-if-you-use-it-for-work-but-youll-have-to-ask-microsoft-for-it-as-a-server-failure-takes-down-outlook-teams-and-exchange/): procedimiento de solicitud en el Centro de administración para la caída de noviembre, nota sobre clientes particulares.

6.  [Office Watch: Microsoft 365 went down? How to claim your SLA credit](https://office-watch.com/?p=78178): exclusión de licencias particulares, gratuitas y para organizaciones sin ánimo de lucro; una solicitud por servicio e incidente.

7.  [Seguimiento en directo: Exchange Online, caída mundial del acceso a buzones](https://rafaelpfister.ch/liveticker/exchange-online-ausfall-2026-11): desarrollo de la incidencia del 12 y 13 de noviembre de 2026 con la cronología de Microsoft.
