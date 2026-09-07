---
title: "Exchange Online limita y bloquea Exchange 2016 y 2019 obsoletos a partir de septiembre de 2026: así funciona el Transport Enforcement"
navTitle: "EXO-Enforcement 09/2026"
description: "A partir de la segunda semana de septiembre de 2026, Exchange Online exige a los servidores híbridos como mínimo la SU de octubre de 2025; de lo contrario, el flujo de correo se limita y posteriormente se bloquea. Contexto sobre el Transport Enforcement desde 2023, las fases de escalado con códigos SMTP, el informe en el Centro de administración, la pausa de 90 días mediante PowerShell y por qué la próxima elevación solo permitirá el acceso a clientes ESU y Exchange SE."
date: "2026-09-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "9 min de lectura"
themen:
  - exchange-onprem-hybrid
  - exchange-updates
produkte:
  - "exchange-hybrid"
  - "exchange-online"
  - "hybrid-mailfluss"
  - "exchange-updates"
protokolle:
  - "smtp"
  - "migration"
  - "releases"
slug: "exchange-online-limita-y-bloquea-exchange-2016-y-2019-obsoletos-a-partir-de-septiembre-de-2026"
translationId: "article-fff0c5efce59ef76"
draft: false
translationOf: exchange-online-transport-enforcement-hybrid-server
url: https://rafaelpfister.ch/es/blog/exchange-online-limita-y-bloquea-exchange-2016-y-2019-obsoletos-a-partir-de-septiembre-de-2026
translationSourceHash: bd79f97bff8aa7047f498355cc9cd65834e03cb24b8d552a841035216f63187c
translationModel: gpt-5.6-terra
translatedAt: 2026-09-07T08:29:00.986Z
translationReview: automatic
---

# Exchange Online limita y bloquea Exchange 2016 y 2019 obsoletos a partir de septiembre de 2026: así funciona el Transport Enforcement

El equipo de Exchange anunció el 2 de septiembre de 2026 que elevará la versión mínima para Exchange 2016 y Exchange 2019 en el flujo de correo híbrido. A partir de la segunda semana de septiembre de 2026, Exchange Online exigirá a los servidores que entregan correo mediante un conector entrante de tipo `OnPremises` como mínimo el nivel de la última actualización de seguridad pública de octubre de 2025. Todo lo que esté por debajo se limitará y posteriormente se bloqueará. En resumen: quien no haya aplicado parches a sus servidores híbridos desde octubre de 2025 perderá gradualmente la entrega de correo a Exchange Online durante las próximas semanas. Y la próxima elevación, que Microsoft prevé para los próximos meses, estará por encima de cualquier actualización disponible públicamente: entonces solo cumplirán el requisito los clientes del programa ESU de pago o los entornos con Exchange Server Subscription Edition (SE).

El anuncio en sí es breve. Lo que significa en la práctica se desprende del sistema de enforcement que Microsoft ha desarrollado gradualmente desde 2023: qué respuestas SMTP verá su servidor, cómo comprobar el estado en el Centro de administración y mediante PowerShell, y qué opciones quedan durante la transición hasta el final del programa ESU en octubre de 2026.

## Lo que se aplica a partir de la segunda semana de septiembre de 2026

El nuevo límite mínimo corresponde a las actualizaciones de seguridad del 14 de octubre de 2025. Fue el último Patch Tuesday en el que Microsoft proporcionó públicamente actualizaciones para Exchange 2016 y 2019; todas las SU desde diciembre de 2025 solo están disponibles a través del programa ESU.

| Versión | Nivel mínimo | KB | Compilación |
|---|---|---|---|
| Exchange 2019 CU15 | SU de octubre de 2025 (CU15 SU5) | KB5066367 | 15.2.1748.39 |
| Exchange 2016 CU23 | SU de octubre de 2025 (CU23 SU19) | KB5066369 | 15.1.2507.61 |

También existe una SU de octubre de 2025 para Exchange 2019 CU14 (KB5066368, compilación 15.2.1544.36). Sin embargo, en la publicación de Microsoft se indica expresamente CU15 SU5 como versión mínima; además, CU14 ya no es un nivel recomendado desde la publicación de CU15 en febrero de 2025. Si utiliza CU14, incluya la actualización a CU15 en la planificación.

Tres delimitaciones son importantes:

- **Solo se ve afectado el flujo de correo híbrido.** Exchange Online comprueba la versión de los servidores que entregan mensajes mediante un conector entrante de tipo `OnPremises`. Esta es la configuración híbrida clásica creada por el Hybrid Configuration Wizard. Los correos que llegan mediante una puerta de enlace de terceros o un conector de tipo `Partner` no pasan por este enforcement.
- **La versión se lee de las cabeceras.** Un servidor Exchange escribe su compilación en la línea `Received` de cada mensaje que reenvía (`… with Microsoft SMTP Server … id 15.1.2507.61`). Exchange Online evalúa esta información. Por tanto, cuenta el nivel del servidor que realmente entrega el mensaje a Exchange Online, es decir, en muchos entornos el Edge Transport Server o los servidores Mailbox con el conector de envío hacia `*.mail.protection.outlook.com`.
- **Exchange SE no se ve afectado.** El enforcement se aplica a Exchange 2016 y 2019; Exchange Server SE se sitúa por encima de cualquier límite mínimo siempre que reciba los parches de forma habitual.

## Contexto: el Transport Enforcement desde 2023

El anuncio de septiembre no es una medida nueva, sino la siguiente fase de un sistema que Microsoft presentó en marzo de 2023 bajo el título «Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online». Microsoft define como «persistently vulnerable» todo servidor Exchange que haya llegado al final de soporte o que permanezca sin parches frente a vulnerabilidades conocidas. El objetivo es proteger a los destinatarios de Exchange Online de mensajes procedentes de servidores que puedan verse comprometidos y, al mismo tiempo, presionar a los operadores para que apliquen parches a los servidores o los desconecten.

El sistema se ha activado por versiones:

| Fecha | Versión afectada |
|---|---|
| Agosto de 2023 | Exchange 2007 |
| Septiembre de 2023 | Exchange 2010 |
| Diciembre de 2023 | Exchange 2013 |
| Marzo de 2024 | Exchange 2016 y 2019 (niveles de SU considerablemente obsoletos) |
| Septiembre de 2026 | Exchange 2016 y 2019: límite mínimo = SU de octubre de 2025 |
| «en unos meses» | Exchange 2016 y 2019: límite mínimo por encima de la última actualización pública |

Hasta ahora, el límite mínimo para Exchange 2016 y 2019 estaba en niveles «significantly behind on security updates». La novedad es que Microsoft sitúa el límite en la última actualización pública y, por tanto, afecta por primera vez a servidores que hace menos de un año todavía estaban completamente actualizados.

## Las fases de escalado

El enforcement funciona con tres funciones que Microsoft denomina «reporting», «throttling» y «blocking». En cuanto un servidor queda por debajo del límite mínimo, comienza un ciclo de 90 días. Las fases de la publicación básica de 2023:

| Periodo | Medida | Respuesta SMTP |
|---|---|---|
| Día 0 a 30 | Solo informe en el Centro de administración de Exchange | ninguna |
| Día 30 a 40 | Limitación 5 minutos por hora | `450 4.7.230` |
| Día 40 a 50 | Limitación 10 minutos por hora | `450 4.7.230` |
| Día 50 a 60 | Limitación 20 minutos por hora | `450 4.7.230` |
| Día 60 a 70 | Limitación 30 minutos por hora, además bloqueo 5 minutos por hora | `450 4.7.230` y `550 5.7.230` |
| Día 70 a 80 | Bloqueo 10 minutos por hora | `550 5.7.230` |
| Día 80 a 90 | Bloqueo 20 minutos por hora | `550 5.7.230` |
| A partir del día 90 | Bloqueo completo | `550 5.7.230` |

Las dos respuestas son literalmente:

```text
450 4.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online throttled for n mins/hr.

550 5.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online blocked for n mins/hr.
```

La diferencia es decisiva para la operación. Con `450`, Exchange Online rechaza temporalmente la conexión; el servidor local conserva el mensaje en su cola e intenta enviarlo de nuevo. Al principio, los usuarios solo notan retrasos; en Queue Viewer o en `Get-Queue` crece la cola hacia el conector de envío a Exchange Online con el estado `Retry` y el mensaje 4.7.230 como `LastError`. Con `550`, el rechazo es definitivo: el remitente recibe un NDR con el código 5.7.230 y el mensaje se pierde si no se vuelve a enviar. Puesto que al principio el bloqueo solo está activo unos minutos por hora, el patrón de error parece esporádico: parte de los mensajes llega y parte falla con NDR. Si observa este patrón en el seguimiento de mensajes, compruebe primero la versión antes de buscar problemas de red o certificados.

El anuncio no indica si Microsoft iniciará el ciclo completo de 90 días para el nuevo límite mínimo a partir de la segunda semana de septiembre o si comenzará ya en una fase posterior. La publicación básica establece que, tras una pausa, el sistema continúa en la fase que había alcanzado previamente. Por tanto, no cuente con 30 días de margen.

## Informe en el Centro de administración de Exchange y mediante PowerShell

Exchange Online enumera los servidores locales detectados junto con su versión en un informe específico: en el Centro de administración de Exchange, en *Reports*, *Mail flow*, informe sobre servidores Exchange locales conectados desactualizados («out-of-date connecting on-premises Exchange servers»). El informe muestra para cada servidor la compilación detectada, si está por debajo del límite mínimo y en qué fase se encuentra el enforcement.

Exchange Online PowerShell proporciona la misma información:

```powershell
Connect-ExchangeOnline
Get-OnPremServerReportInfo
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Comando | Efecto |
|---|---|
| `Connect-ExchangeOnline` | Abre la sesión con Exchange Online (módulo `ExchangeOnlineManagement`). |
| `Get-OnPremServerReportInfo` | Enumera los servidores locales detectados por Exchange Online con compilación, estado de enforcement y fase. |

</details>

El informe solo conoce los servidores que realmente entregan correo a Exchange Online. Un servidor de administración sin flujo de correo o una máquina que solo tenga Management Tools no aparece. Esto es irrelevante para el enforcement, pero no para la seguridad: estos sistemas también necesitan las SU.

## Pausar el enforcement: 90 días al año

Para los entornos que no puedan alcanzar el límite mínimo a corto plazo, Microsoft ofrece una pausa. Puede activarse por un total de 90 días al año, de forma continua o en varios periodos:

```powershell
Get-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer

New-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer `
  -NumberOfDays 30
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Get-TenantExemptionInfo` | Muestra si hay una pausa activa para el tenant y cuánto tiempo queda. |
| `New-TenantExemptionInfo` | Crea una nueva pausa. |
| `-BlockingScenario UnpatchedOnPremServer` | Selecciona el escenario «servidor local desactualizado»; actualmente no existen otros escenarios para este cmdlet. |
| `-NumberOfDays 30` | Duración de la pausa en días. El cupo es de 90 días al año y se descuenta de él el valor indicado. |

</details>

Dos características de la pausa son importantes en la práctica. En primer lugar, al finalizar, el enforcement continúa en la fase en la que se detuvo; la pausa no reinicia el ciclo de 90 días. En segundo lugar, no hay ningún cmdlet para finalizar anticipadamente una pausa en curso: quien cree una pausa de 90 días y termine de aplicar parches dos semanas después habrá agotado el cupo anual. Por tanto, cree la pausa por el menor tiempo posible y amplíela si es necesario.

Además, la pausa solo es una solución para el límite mínimo actual. Si Microsoft eleva el límite dentro de unos meses por encima de la última actualización pública, un cupo agotado ya no ayudará.

## Por qué la próxima elevación es el verdadero plazo

Exchange 2016 y 2019 están fuera de soporte desde el 14 de octubre de 2025. Posteriormente, Microsoft lanzó dos periodos ESU de pago: el periodo 1 hasta abril de 2026 y el periodo 2 de mayo a octubre de 2026. Al anunciar el periodo 2 el 15 de abril de 2026, el equipo de Exchange aclaró que no habrá más ampliaciones. Las SU de diciembre de 2025 a agosto de 2026 (las últimas, compilación 15.2.1748.49 para 2019 CU15 y 15.1.2507.72 para 2016 CU23) están disponibles exclusivamente para clientes ESU y no se ofrecen públicamente para su descarga.

De ello resulta la siguiente situación:

- **Hoy**, un servidor con la SU de octubre de 2025 cumple el límite mínimo, con o sin ESU.
- **Con la próxima elevación**, según Microsoft, el límite mínimo estará por encima del nivel de octubre de 2025. Sin contrato ESU, no existe una vía legal para alcanzar ese nivel. El flujo de correo híbrido de esos servidores se limitará y bloqueará entonces, independientemente de lo bien gestionado que esté el resto del entorno.
- **El 31 de octubre de 2026** también finaliza el periodo 2. Después ya no habrá SU para Exchange 2016 y 2019, para nadie. Por tanto, como muy tarde la siguiente elevación posterior también afectará a los clientes ESU.

Así, el programa ESU compra, en el mejor de los casos, unos pocos meses. El único nivel permanente que permite el enforcement es Exchange Server SE. Microsoft también ha anunciado que Exchange SE CU2, previsto para la segunda mitad de 2026, terminará la coexistencia con Exchange 2016 y 2019: la instalación se interrumpirá si encuentra servidores antiguos en la organización. Por ello, la migración no solo es necesaria por el flujo de correo, sino también para poder seguir instalando actualizaciones de SE.

Para los entornos que solo conservan Exchange On-Premises para administrar atributos en una configuración híbrida, la alternativa es eliminar el último servidor: desde Exchange 2019 CU12, los atributos de destinatarios pueden mantenerse con las Management Tools sin un servidor Exchange en ejecución. En ese caso, ya no hay flujo de correo local y el enforcement deja de ser relevante.

## Determinar la versión

`Get-ExchangeServer` muestra en `AdminDisplayVersion` solo la CU, no la SU. Es fiable comprobar la versión del archivo `ExSetup.exe` o usar el [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), que además informa de los pasos manuales pendientes. Para obtener una vista rápida de todos los servidores:

```powershell
Get-ExchangeServer | ForEach-Object {
  $path = "\\$($_.Name)\C$\Program Files\Microsoft\Exchange Server\V15\bin\ExSetup.exe"
  [pscustomobject]@{
    Server  = $_.Name
    Version = (Get-Item $path).VersionInfo.ProductVersion
  }
}
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Elemento | Efecto |
|---|---|
| `Get-ExchangeServer` | Enumera todos los servidores Exchange de la organización. |
| `\\<Server>\C$\...\ExSetup.exe` | Ruta administrativa compartida al archivo de configuración; adáptela si la ruta de instalación es distinta. |
| `VersionInfo.ProductVersion` | Versión del archivo que corresponde a la compilación SU instalada (p. ej., `15.1.2507.61`). |

</details>

Si la versión está por debajo de `15.2.1748.39` (2019 CU15) o de `15.1.2507.61` (2016 CU23), el servidor quedará por debajo del límite mínimo a partir de la segunda semana de septiembre.

## Procedimiento recomendado

1. **Inventaríe el estado** como se ha descrito anteriormente, incluidos los Edge Transport Server y los servidores de administración.

2. **Compruebe el informe en Exchange Online.** `Get-OnPremServerReportInfo` muestra qué servidores ve realmente Exchange Online y si ya está activa una fase de enforcement. Compare la lista con el inventario: los servidores que no aparecen ahí no entregan correo mediante el conector `OnPremises`.

3. **Instale al menos la SU de octubre de 2025.** KB5066367 (2019 CU15) y KB5066369 (2016 CU23) siguen disponibles públicamente en el Microsoft Download Center. Las SU son acumulativas; un servidor en el nivel de agosto de 2025 puede actualizarse directamente a octubre de 2025. En CU14, instale primero CU15. Después de la instalación, reinicie, compruebe el estado de los servicios y ejecute de nuevo el Health Checker.

4. **Use la pausa solo como medida transitoria.** Si no logra instalar la actualización durante la primera mitad de septiembre, cree `New-TenantExemptionInfo` con una duración breve y no considere la pausa como una reserva de planificación para la próxima elevación.

5. **Programe la migración a Exchange SE.** Sin contrato ESU, la próxima elevación es el plazo definitivo; con ESU, lo es el 31 de octubre de 2026. Exchange 2019 CU15 puede actualizarse a SE mediante una actualización in situ; Exchange 2016 requiere pasar por una instalación nueva de SE y el traslado de buzones o funciones. Quien solo opere Exchange para la administración de atributos puede eliminar el último servidor y continuar trabajando con las Management Tools.

## Fuentes

1.  [Exchange 2016/2019: Throttling and Blocking up to the Final Public Update Baseline – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-20162019-throttling-and-blocking-up-to-the-final-public-update-baseline/4552717): El anuncio del 2 de septiembre de 2026 con el nuevo límite mínimo (SU de octubre de 2025), la fecha de inicio en la segunda semana de septiembre y la indicación de la próxima elevación por encima de la última actualización pública.

2.  [Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/throttling-and-blocking-email-from-persistently-vulnerable-exchange-servers-to-e/3815328): La publicación básica de 2023 con la definición de «persistently vulnerable», las fases Reporting, Throttling y Blocking, el ciclo de 90 días, las respuestas SMTP 4.7.230 y 5.7.230, así como el plan de implementación por versiones.

3.  [How to pause throttling and blocking of out-of-date on-premises Exchange Servers – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/how-to-pause-throttling-and-blocking-of-out-of-date-on-premises-exchange-servers/4007169): Los cmdlets `Get-OnPremServerReportInfo`, `Get-TenantExemptionInfo` y `New-TenantExemptionInfo`, junto con el informe en el Centro de administración de Exchange y el cupo anual de 90 días.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Números de compilación de las SU de octubre de 2025 y de las posteriores actualizaciones ESU hasta agosto de 2026; también incluye la indicación de que las SU desde diciembre de 2025 solo las reciben los clientes ESU.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU15: October 14, 2025 (KB5066367) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2019-cu15-october-14-2025-kb5066367-19a1091c-e0b3-4078-be6b-312463063d05): El artículo KB sobre el nivel mínimo para Exchange 2019 CU15.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: October 14, 2025 (KB5066369) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2016-cu23-october-14-2025-kb5066369-8ca2ab99-dfde-4329-896a-3faa677d2603): El artículo KB sobre el nivel mínimo para Exchange 2016 CU23.

7.  [Support for Exchange Server 2016 and Exchange Server 2019 ends today – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/support-for-exchange-server-2016-and-exchange-server-2019-ends-today/4461192): El final de soporte el 14 de octubre de 2025.

8.  [Announcing Exchange 2016 / 2019 Extended Security Update program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-exchange-2016--2019-extended-security-update-program/4433495): Condiciones del primer periodo ESU.

9.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Periodo de mayo a octubre de 2026 y la afirmación de que no habrá más ampliaciones.

10. [Exchange Online transport enforcement system explained – CodeTwo Admin's Blog](https://www.codetwo.com/admins-blog/persistently-vulnerable-exchange-server/): Desglose tabular de las ocho fases de enforcement y las fechas de implementación para cada versión de Exchange; fuente de terceros.

11. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventario de niveles CU/SU y pasos manuales pendientes.
