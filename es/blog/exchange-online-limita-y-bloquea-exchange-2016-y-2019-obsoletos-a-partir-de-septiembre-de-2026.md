---
title: "Exchange Online limita y bloquea servidores Exchange 2016 y 2019 obsoletos a partir de septiembre de 2026: así funciona la aplicación de transporte"
navTitle: "Aplicación EXO 09/2026"
description: "A partir de la segunda semana de septiembre de 2026, Exchange Online exigirá a los servidores híbridos al menos la SU de octubre de 2025; de lo contrario, el flujo de correo se limitará y posteriormente se bloqueará. Contexto sobre la aplicación de transporte desde 2023, las fases de escalada con códigos SMTP, el informe en el Centro de administración, la pausa de 90 días mediante PowerShell y por qué el próximo aumento solo permitirá el acceso a clientes de ESU y Exchange SE."
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
translationSourceHash: d2bb7589193052361cd6a083ace0d3dbc58552e5e66691c352cd82b693fae6bd
translationModel: gpt-5.6-terra
translatedAt: 2026-10-28T09:50:27.989Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/exchange-online-limita-y-bloquea-exchange-2016-y-2019-obsoletos-a-partir-de-septiembre-de-2026
---

# Exchange Online limita y bloquea servidores Exchange 2016 y 2019 obsoletos a partir de septiembre de 2026: así funciona la aplicación de transporte

El equipo de Exchange anunció el 2 de septiembre de 2026 que elevará la versión mínima para Exchange 2016 y Exchange 2019 en el flujo de correo híbrido. A partir de la segunda semana de septiembre de 2026, Exchange Online exigirá a los servidores que entreguen correo mediante un conector de entrada de tipo `OnPremises` al menos el nivel de la última actualización pública de seguridad de octubre de 2025. Todo lo que esté por debajo se limitará y posteriormente se bloqueará. En resumen: quien no haya aplicado parches a sus servidores híbridos desde octubre de 2025 perderá progresivamente, en las próximas semanas, la entrega de correo a Exchange Online. Además, el próximo aumento que Microsoft prevé para los próximos meses estará por encima de cualquier actualización disponible públicamente: entonces, solo los clientes del programa ESU de pago o los entornos con Exchange Server Subscription Edition (SE) cumplirán el requisito.

El anuncio en sí es breve. Lo que significa en la práctica se desprende del sistema de aplicación que Microsoft ha ido creando gradualmente desde 2023: qué respuestas SMTP verá su servidor, cómo comprobar el estado en el Centro de administración y mediante PowerShell, y qué opciones quedan durante la transición hasta el final del programa ESU en octubre de 2026.

## Lo que se aplica a partir de la segunda semana de septiembre de 2026

El nuevo límite mínimo corresponde a las actualizaciones de seguridad del 14 de octubre de 2025. Fue el último Patch Tuesday en el que Microsoft proporcionó actualizaciones públicas para Exchange 2016 y 2019; todas las SU desde diciembre de 2025 solo están disponibles mediante el programa ESU.

| Versión | Nivel mínimo | KB | Compilación |
|---|---|---|---|
| Exchange 2019 CU15 | SU de octubre de 2025 (CU15 SU5) | KB5066367 | 15.2.1748.39 |
| Exchange 2016 CU23 | SU de octubre de 2025 (CU23 SU19) | KB5066369 | 15.1.2507.61 |

También existe una SU de octubre de 2025 para Exchange 2019 CU14 (KB5066368, compilación 15.2.1544.36). Sin embargo, la publicación de Microsoft indica expresamente CU15 SU5 como versión mínima; CU14 ya no es un nivel recomendado desde la publicación de CU15 en febrero de 2025. Si utiliza CU14, incluya en su planificación el salto a CU15.

Tres delimitaciones son importantes:

- **Solo se ve afectado el flujo de correo híbrido.** Exchange Online comprueba la versión de los servidores de entrega para los mensajes que llegan mediante un conector de entrada de tipo `OnPremises`. Esta es la configuración híbrida clásica que crea el Asistente para configuración híbrida. Los correos que llegan a través de una puerta de enlace de terceros o de un conector de tipo `Partner` no pasan por esta aplicación.
- **La versión se lee de las cabeceras.** Un servidor Exchange escribe su compilación en la línea `Received` de cada mensaje que reenvía (`… with Microsoft SMTP Server … id 15.1.2507.61`). Exchange Online evalúa este dato. Por tanto, cuenta el nivel del servidor que realmente entrega el mensaje a Exchange Online; es decir, en muchos entornos, el servidor Edge Transport o el servidor de buzones con el conector de envío hacia `*.mail.protection.outlook.com`.
- **Exchange SE no se ve afectado.** La aplicación afecta a Exchange 2016 y 2019; Exchange Server SE está por encima de cualquier límite mínimo siempre que se le apliquen parches periódicamente.

## Antecedentes: la aplicación de transporte desde 2023

El anuncio de septiembre no es una medida nueva, sino la siguiente fase de un sistema que Microsoft presentó en marzo de 2023 bajo el título «Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online». Microsoft define como «persistently vulnerable» cualquier servidor Exchange que haya llegado al final del soporte o permanezca sin parches para vulnerabilidades conocidas. El objetivo es proteger a los destinatarios de Exchange Online frente a mensajes procedentes de servidores susceptibles de ser comprometidos y, al mismo tiempo, presionar a los operadores para que apliquen parches a los servidores o los apaguen.

El sistema se activó por versiones:

| Momento | Versión afectada |
|---|---|
| Agosto de 2023 | Exchange 2007 |
| Septiembre de 2023 | Exchange 2010 |
| Diciembre de 2023 | Exchange 2013 |
| Marzo de 2024 | Exchange 2016 y 2019 (niveles de SU considerablemente obsoletos) |
| Septiembre de 2026 | Exchange 2016 y 2019: límite mínimo = SU de octubre de 2025 |
| «en algunos meses» | Exchange 2016 y 2019: límite mínimo por encima de la última actualización pública |

Para Exchange 2016 y 2019, el límite mínimo se situaba hasta ahora en niveles que estaban «significantly behind on security updates». La novedad es que Microsoft eleva el límite hasta la última actualización pública y, con ello, afecta por primera vez a servidores que hace menos de un año aún estaban completamente actualizados.

## Las fases de escalada

La aplicación funciona en tres modalidades, que Microsoft denomina «reporting», «throttling» y «blocking». En cuanto un servidor cae por debajo del límite mínimo, comienza un ciclo de 90 días. Las fases de la publicación básica de 2023:

| Periodo | Medida | Respuesta SMTP |
|---|---|---|
| Día 0 a 30 | Solo informe en el Centro de administración de Exchange | ninguna |
| Día 30 a 40 | Limitación durante 5 minutos por hora | `450 4.7.230` |
| Día 40 a 50 | Limitación durante 10 minutos por hora | `450 4.7.230` |
| Día 50 a 60 | Limitación durante 20 minutos por hora | `450 4.7.230` |
| Día 60 a 70 | Limitación durante 30 minutos por hora y, adicionalmente, bloqueo durante 5 minutos por hora | `450 4.7.230` y `550 5.7.230` |
| Día 70 a 80 | Bloqueo durante 10 minutos por hora | `550 5.7.230` |
| Día 80 a 90 | Bloqueo durante 20 minutos por hora | `550 5.7.230` |
| Desde el día 90 | Bloqueo completo | `550 5.7.230` |

Las dos respuestas son, literalmente:

```text
450 4.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online throttled for n mins/hr.

550 5.7.230 Connecting Exchange server version is out-of-date;
connection to Exchange Online blocked for n mins/hr.
```

La diferencia es decisiva para la operación. Con `450`, Exchange Online rechaza temporalmente la conexión; el servidor local conserva el mensaje en su cola e intenta enviarlo de nuevo. Al principio, los usuarios solo notan retrasos; en Queue Viewer o en `Get-Queue` crece la cola del conector de envío hacia Exchange Online con el estado `Retry` y el mensaje 4.7.230 como `LastError`. Con `550`, el rechazo es definitivo: el remitente recibe un NDR con el código 5.7.230 y el mensaje se pierde, salvo que se envíe de nuevo. Dado que al principio el bloqueo solo está activo algunos minutos por hora, el patrón de error parece inicialmente esporádico: parte de los mensajes llega y parte falla con NDR. Quien vea un patrón así en el seguimiento de mensajes debería comprobar primero la versión antes de buscar problemas de red o certificados.

El anuncio no indica si Microsoft iniciará el ciclo completo de 90 días para el nuevo límite mínimo a partir de la segunda semana de septiembre o si comenzará ya en una fase posterior. La publicación básica establece que el sistema continúa tras una pausa en la fase alcanzada previamente. Por tanto, no cuente con 30 días de margen.

## Informe en el Centro de administración de Exchange y mediante PowerShell

Exchange Online enumera los servidores locales detectados, junto con su versión, en un informe específico: en el Centro de administración de Exchange, en *Reports*, *Mail flow*, informe sobre servidores Exchange locales conectados obsoletos («out-of-date connecting on-premises Exchange servers»). El informe muestra por servidor la compilación detectada, si está por debajo del límite mínimo y en qué fase se encuentra la aplicación.

Exchange Online PowerShell proporciona la misma información:

```powershell
Connect-ExchangeOnline
Get-OnPremServerReportInfo
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Comando | Efecto |
|---|---|
| `Connect-ExchangeOnline` | Abre la sesión con Exchange Online (módulo `ExchangeOnlineManagement`). |
| `Get-OnPremServerReportInfo` | Enumera los servidores locales detectados por Exchange Online con compilación, estado y fase de aplicación. |

</details>

El informe solo conoce los servidores que realmente entregan correo a Exchange Online. Un servidor de administración sin flujo de correo o una máquina que solo tenga Management Tools no aparecerá. Esto es irrelevante para la aplicación, pero no para la seguridad: esos sistemas también necesitan las SU.

## Pausar la aplicación: 90 días por año

Para entornos que no pueden alcanzar el límite mínimo a corto plazo, Microsoft ofrece una pausa. Puede activarse por un total de 90 días al año, de una vez o en varios periodos:

```powershell
Get-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer

New-TenantExemptionInfo -BlockingScenario UnpatchedOnPremServer `
  -NumberOfDays 30
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `Get-TenantExemptionInfo` | Muestra si hay una pausa activa para el tenant y su duración. |
| `New-TenantExemptionInfo` | Crea una pausa nueva. |
| `-BlockingScenario UnpatchedOnPremServer` | Selecciona el escenario «servidor local obsoleto»; actualmente no existen otros escenarios para este cmdlet. |
| `-NumberOfDays 30` | Duración de la pausa en días. El cupo es de 90 días al año y el valor se descuenta de él. |

</details>

Dos características de la pausa son importantes en la práctica. En primer lugar, la aplicación continúa tras su vencimiento en la fase en la que se detuvo; la pausa no reinicia el ciclo de 90 días. En segundo lugar, no hay ningún cmdlet para finalizar anticipadamente una pausa en curso: quien establezca 90 días y termine de aplicar los parches tras dos semanas habrá agotado el cupo anual. Por ello, establezca la pausa durante el menor tiempo posible y amplíela si es necesario.

Además, la pausa solo es una solución para el límite mínimo actual. Si Microsoft eleva el límite en algunos meses por encima de la última actualización pública, un cupo agotado ya no ayudará.

## Por qué el próximo aumento es el plazo real

Exchange 2016 y 2019 están fuera de soporte desde el 14 de octubre de 2025. Posteriormente, Microsoft lanzó dos periodos ESU de pago: el periodo 1 hasta abril de 2026 y el periodo 2 de mayo a octubre de 2026. Con el anuncio del periodo 2 el 15 de abril de 2026, el equipo de Exchange dejó claro que no habrá más ampliaciones. Las SU de diciembre de 2025 a octubre de 2026 (por último, la compilación 15.2.1748.55 para 2019 CU15 y 15.1.2507.77 para 2016 CU23, ambas del 13 de octubre de 2026) están disponibles exclusivamente para clientes ESU y no se ofrecen públicamente para su descarga.

De ello se deriva la siguiente situación:

- **Hoy**, un servidor con la SU de octubre de 2025 cumple el límite mínimo, con o sin ESU.
- **Con el próximo aumento**, el límite mínimo estará, según Microsoft, por encima del nivel de octubre de 2025. Sin un contrato ESU, no hay ninguna forma legal de alcanzar ese nivel. El flujo de correo híbrido de esos servidores se limitará y bloqueará, independientemente de lo bien que se gestione el resto del entorno.
- **El 31 de octubre de 2026** también finaliza el periodo 2. Microsoft publicó las últimas actualizaciones para Exchange 2016 y 2019 con la SU de octubre del 13 de octubre de 2026; después ya no habrá más SU para nadie. Por tanto, como muy tarde el aumento siguiente afectará también a los clientes ESU.

Por tanto, el programa ESU compra, en el mejor de los casos, unos pocos meses. La única plataforma permanente que permite la aplicación es Exchange Server SE. Microsoft también ha anunciado que Exchange SE CU2 (originalmente previsto para la segunda mitad de 2026; dado que Microsoft no indica actualmente ninguna fecha para el CU1 anterior, tampoco hay fecha para CU2) finalizará la coexistencia con Exchange 2016 y 2019: la instalación se cancelará si encuentra servidores antiguos en la organización. Por ello, la migración no solo es necesaria por el flujo de correo, sino también para poder seguir instalando actualizaciones para SE.

Para entornos que conservan Exchange On-Premises únicamente para administrar atributos en una configuración híbrida, la alternativa es eliminar el último servidor: desde Exchange 2019 CU12, los atributos de destinatario pueden mantenerse con Management Tools sin un servidor Exchange en ejecución. En ese caso, ya no existe flujo de correo local y la aplicación deja de ser relevante.

## Determinar la versión

`Get-ExchangeServer` muestra en `AdminDisplayVersion` solo la CU, no la SU. Es fiable la versión de archivo de `ExSetup.exe` o el [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), que además informa de los pasos manuales pendientes. Para obtener una visión rápida de todos los servidores:

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
<summary>Explicación de las opciones</summary>

| Elemento | Efecto |
|---|---|
| `Get-ExchangeServer` | Enumera todos los servidores Exchange de la organización. |
| `\\<Server>\C$\...\ExSetup.exe` | Ruta de recurso compartido administrativo al archivo de instalación; adáptela si la ruta de instalación es distinta. |
| `VersionInfo.ProductVersion` | Versión del archivo, que corresponde a la compilación de SU instalada (p. ej., `15.1.2507.61`). |

</details>

Si la versión es inferior a `15.2.1748.39` (2019 CU15) o `15.1.2507.61` (2016 CU23), el servidor estará por debajo del límite mínimo a partir de la segunda semana de septiembre.

## Procedimiento recomendado

1. **Inventarie el estado** como se ha descrito anteriormente, incluidos los servidores Edge Transport y los servidores de administración.

2. **Compruebe el informe en Exchange Online.** `Get-OnPremServerReportInfo` muestra qué servidores ve realmente Exchange Online y si ya está activa una fase de aplicación. Compare la lista con el inventario: los servidores que faltan allí no entregan correo mediante el conector `OnPremises`.

3. **Instale como mínimo la SU de octubre de 2025.** KB5066367 (2019 CU15) y KB5066369 (2016 CU23) siguen estando disponibles públicamente en Microsoft Download Center. Las SU son acumulativas; un servidor en el nivel de agosto de 2025 puede actualizarse directamente a octubre de 2025. En CU14, instale primero CU15. Tras la instalación, reinicie, compruebe el estado de los servicios y vuelva a ejecutar Health Checker.

4. **Use la pausa solo como medida transitoria.** Si no es posible completar la actualización en la primera mitad de septiembre, cree `New-TenantExemptionInfo` con una duración breve y no considere la pausa una reserva de planificación para el próximo aumento.

5. **Programe la migración a Exchange SE.** Sin contrato ESU, el próximo aumento es el plazo inaplazable; con ESU, lo es el 31 de octubre de 2026. Exchange 2019 CU15 puede actualizarse a SE mediante una actualización in situ; Exchange 2016 requiere el paso intermedio de una instalación nueva de SE y el traslado de buzones o roles. Quien opere Exchange únicamente para administrar atributos debe eliminar el último servidor y continuar trabajando con Management Tools.

## Historial de cambios

- **25 de octubre de 2026:** Añadido: las SU de octubre del 13 de octubre de 2026 (compilaciones 15.2.1748.55 y 15.1.2507.77) son las últimas actualizaciones para Exchange 2016 y 2019. No hay fecha para Exchange SE CU2 debido al aplazamiento de CU1.

## Fuentes

1.  [Exchange 2016/2019: Throttling and Blocking up to the Final Public Update Baseline – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-20162019-throttling-and-blocking-up-to-the-final-public-update-baseline/4552717): El anuncio del 2 de septiembre de 2026 con el nuevo límite mínimo (SU de octubre de 2025), la fecha de inicio en la segunda semana de septiembre y la referencia al próximo aumento por encima de la última actualización pública.

2.  [Throttling and Blocking Email from Persistently Vulnerable Exchange Servers to Exchange Online – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/throttling-and-blocking-email-from-persistently-vulnerable-exchange-servers-to-e/3815328): La publicación básica de 2023 con la definición de «persistently vulnerable», las fases Reporting, Throttling y Blocking, el ciclo de 90 días, las respuestas SMTP 4.7.230 y 5.7.230, así como el plan de implementación por versiones.

3.  [How to pause throttling and blocking of out-of-date on-premises Exchange Servers – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/how-to-pause-throttling-and-blocking-of-out-of-date-on-premises-exchange-servers/4007169): Los cmdlets `Get-OnPremServerReportInfo`, `Get-TenantExemptionInfo` y `New-TenantExemptionInfo`, junto con el informe en el Centro de administración de Exchange y el cupo anual de 90 días.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Números de compilación de las SU de octubre de 2025 y las actualizaciones ESU posteriores hasta octubre de 2026; también incluye la indicación de que las SU a partir de diciembre de 2025 solo las reciben clientes ESU.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU15: October 14, 2025 (KB5066367) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2019-cu15-october-14-2025-kb5066367-19a1091c-e0b3-4078-be6b-312463063d05): El artículo KB sobre el nivel mínimo para Exchange 2019 CU15.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: October 14, 2025 (KB5066369) – Microsoft Support](https://support.microsoft.com/en-us/topic/description-of-the-security-update-for-microsoft-exchange-server-2016-cu23-october-14-2025-kb5066369-8ca2ab99-dfde-4329-896a-3faa677d2603): El artículo KB sobre el nivel mínimo para Exchange 2016 CU23.

7.  [Support for Exchange Server 2016 and Exchange Server 2019 ends today – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/support-for-exchange-server-2016-and-exchange-server-2019-ends-today/4461192): El fin del soporte el 14 de octubre de 2025.

8.  [Announcing Exchange 2016 / 2019 Extended Security Update program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-exchange-2016--2019-extended-security-update-program/4433495): Condiciones del primer periodo ESU.

9.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Vigencia de mayo a octubre de 2026, la afirmación de que no habrá más ampliación y la actualización del 13 de octubre de 2026 sobre las últimas actualizaciones para Exchange 2016 y 2019.

10. [Exchange Online transport enforcement system explained – CodeTwo Admin's Blog](https://www.codetwo.com/admins-blog/persistently-vulnerable-exchange-server/): Desglose tabular de las ocho fases de aplicación y las fechas de implementación para cada versión de Exchange; fuente de terceros.

11. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventario de niveles CU/SU y pasos manuales pendientes.

12. [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837): El equipo de Exchange sobre el aplazamiento de Exchange SE CU1 sin fecha objetivo.
