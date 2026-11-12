---
title: "Actualizaciones de seguridad de Exchange de noviembre de 2026: nueve vulnerabilidades, la primera actualización solo para Exchange SE"
navTitle: "Exchange SU 11/2026"
description: "La SU de noviembre corrige nueve vulnerabilidades en Exchange Server SE, incluida una ejecución remota de código crítica para la que basta una cuenta con pocos privilegios. Exchange 2016 y 2019 ya no reciben actualizaciones. La actualización de Windows de noviembre corrige las interrupciones de Exchange tras las actualizaciones de Windows de octubre."
date: "2026-11-11"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min de lectura"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
slug: "actualizaciones-de-seguridad-de-exchange-de-noviembre-de-2026-nueve-vulnerabilidades-la-primera"
translationId: article-cfb739155e545cc3
translationOf: exchange-security-updates-november-2026
url: https://rafaelpfister.ch/es/blog/actualizaciones-de-seguridad-de-exchange-de-noviembre-de-2026-nueve-vulnerabilidades-la-primera
translationSourceHash: 1ce238b3219fc9c3670ec3137e807538c62e566ad22bd60fbee45dcf6dce9193
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:24:49.322Z
translationReview: automatic
---

# Actualizaciones de seguridad de Exchange de noviembre de 2026: nueve vulnerabilidades, la primera actualización solo para Exchange SE

Microsoft publicó el 10 de noviembre de 2026 una actualización de seguridad (SU) para Exchange Server SE. Según el artículo de KB, corrige nueve vulnerabilidades. Ninguna de ellas se conocía públicamente de antemano y, según Security Update Guide, ninguna se está explotando activamente. Microsoft clasifica una ejecución remota de código como *Critical* y «Exploitation More Likely»; para ella basta una cuenta con pocos privilegios. Este mes es especial por dos motivos: es la primera SU que se publica únicamente para Exchange SE y las interrupciones de Exchange tras las actualizaciones de Windows Server de octubre se han corregido con las actualizaciones de Windows de noviembre.

## Para qué versiones de Exchange está disponible la actualización

La SU del 10 de noviembre de 2026 solo está disponible para una versión:

- **Exchange Server Subscription Edition (SE) RTM**: KB5133453, compilación 15.2.2562.57; disponible públicamente.

Puede comparar su versión con el resumen de [números de compilación de Exchange](/tools/exchange-builds).

No hay actualización para Exchange 2016 ni 2019. La [SU de octubre](/blog/exchange-security-updates-oktober-2026) fue la última, también para clientes del programa ESU de período 2, y según el anuncio dicho programa ha finalizado. Según Microsoft, Exchange Online ya está protegido; sin embargo, en entornos híbridos todos los servidores Exchange necesitan la SU, incluidos los servidores dedicados exclusivamente a administración y los equipos con Exchange Management Tools.

## Resumen de las vulnerabilidades

| CVE | Tipo | CVSS |
| --- | --- | --- |
| CVE-2026-118207 | Ejecución remota de código | 8.8 |
| CVE-2026-118149 | Ejecución remota de código | 8.8 |
| CVE-2026-118150 | Elevación de privilegios | 8.8 |
| CVE-2026-118154 | Elevación de privilegios | 8.8 |
| CVE-2026-118153 | Denegación de servicio | 7.5 |
| CVE-2026-118151 | Elevación de privilegios | 7.2 |
| CVE-2026-118157 | Elevación de privilegios | 7.2 |
| CVE-2026-118208 | Suplantación de identidad | 7.1 |
| CVE-2026-118152 | Suplantación de identidad | 6.5 |

Las nueve vulnerabilidades son accesibles a través de la red (AV:N) y, según el vector CVSS, tienen una complejidad de ataque baja (AC:L). Security Update Guide solo incluye Exchange Server SE RTM como producto afectado.

**[CVE-2026-118207](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118207) es la única vulnerabilidad del mes que Microsoft clasifica como *Critical* (ejecución remota de código, CVSS 8.8) y la única con la valoración «Exploitation More Likely». Según las preguntas frecuentes de Security Update Guide, un atacante autenticado con pocos privilegios puede enviar una solicitud preparada a un servidor afectado y ejecutar código en él; no se requiere interacción del usuario. Microsoft indica como causa una validación de entrada insuficiente (CWE-706). Por tanto, para el ataque basta una cuenta de buzón ordinaria, por ejemplo, una cuya contraseña haya sido obtenida mediante phishing.

**[CVE-2026-118149](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118149) es la segunda ejecución remota de código (CVSS 8.8, *Important*). También en este caso, según las preguntas frecuentes, un atacante autenticado envía una solicitud preparada a un servicio afectado y después puede ejecutar código en el sistema de destino; la causa es una inyección de código (CWE-94). Microsoft valora la explotación como «Exploitation Unlikely».

**[CVE-2026-118153](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118153) (denegación de servicio, CVSS 7.5) es la única vulnerabilidad que no requiere ni una cuenta ni interacción del usuario. Según la descripción, un atacante no autenticado puede dejar el servicio fuera de funcionamiento a través de la red; la causa es una complejidad algorítmica ineficiente (CWE-407). Según el vector, solo se ve afectada la disponibilidad, no la confidencialidad ni la integridad.

Las cuatro elevaciones de privilegios se diferencian en el requisito: para [CVE-2026-118150](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118150) y [CVE-2026-118154](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118154) (ambas CVSS 8.8) basta una cuenta con pocos privilegios; para [CVE-2026-118151](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118151) y [CVE-2026-118157](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118157) (ambas CVSS 7.2), el atacante ya necesita privilegios elevados. En CVE-2026-118151, según las preguntas frecuentes, puede obtener privilegios SYSTEM.

De las dos vulnerabilidades de suplantación de identidad, [CVE-2026-118208](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118208) (CVSS 7.1) requiere una cuenta con pocos privilegios. [CVE-2026-118152](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-118152) (CVSS 6.5) es una falsificación de solicitud entre sitios: el atacante no necesita una cuenta, pero un usuario debe participar (UI:R). Todas las vulnerabilidades, excepto CVE-2026-118207 y CVE-2026-118149, están clasificadas como «Exploitation Less Likely».

## Exchange 2016 y 2019: sin actualización ni más pruebas

El anuncio menciona Exchange 2016 y 2019 únicamente en una nota: estas versiones ya no se prueban para detectar vulnerabilidades y no se publicarán más actualizaciones. Microsoft escribe que deben considerarse permanentemente vulnerables («permanently vulnerable») y que la organización debe actualizarse de inmediato. Que los nueve CVE de noviembre solo incluyan Exchange SE no dice nada sobre si las versiones antiguas están afectadas. Frankys Web señala que Exchange SE y 2019 CU15 son técnicamente similares y que, probablemente, al menos parte de las vulnerabilidades también está presente en Exchange 2019.

Para la operación, esto significa lo siguiente:

1. **Exchange 2019 CU15 y anteriores, Exchange 2016**: La versión queda congelada en la SU de octubre (con ESU) o una anterior. Las nueve vulnerabilidades de noviembre permanecen sin corregir en estos servidores, si existen en ellos, y Microsoft ya no lo comprobará.

2. **Migración**: Microsoft recomienda cambiar inmediatamente a Exchange SE.

3. **Entornos híbridos**: Exchange Online limita y bloquea el flujo de correo procedente de servidores locales obsoletos; las reglas se describen en el [artículo sobre la aplicación del transporte](/blog/exchange-online-transport-enforcement-hybrid-server).

## Problemas corregidos

**Interrupciones de Exchange tras las actualizaciones de Windows Server de octubre** (todas las versiones). El problema con `MapiExceptionMdbOffline` (ec=1142) no lo corrige la SU de Exchange: según el anuncio, deben instalarse las actualizaciones de Windows Server de noviembre de 2026. Las notas de la versión de estas actualizaciones incluyen la corrección bajo «[Exchange Server]» e indican que el problema está resuelto con actualizaciones a partir del 10 de noviembre de 2026: KB5131080 para Windows Server 2025, KB5131090 para Windows Server 2022, KB5131085 para Windows Server 2019 y KB5131291 para Windows Server 2016. Según el anuncio, el orden en que instale la SU de Exchange y la actualización de Windows no importa. Si configuró el valor de registro `EpmapSavedContextCap` del [artículo sobre las interrupciones de Exchange](/blog/exchange-windows-updates-oktober-2026), puede dejarlo; según Microsoft, ya no es necesario después de instalar las actualizaciones de noviembre para Windows Server.

**Visualización incorrecta de libre/ocupado mediante Microsoft Graph en entornos híbridos** (solo Exchange SE). Según [KB5135867](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5135867), los usuarios con un buzón local no veían información correcta de libre/ocupado para buzones de Exchange Online en el Outlook clásico si Graph API estaba activada; solo se mostraban bloques de tiempo. Con Exchange Web Services (EWS) en lugar de Graph API, la visualización era correcta. El problema se corrige con KB5133453 o una actualización posterior. El artículo de soporte no menciona ninguna acción posterior. Si volvió a EWS debido a este error, compruebe tras la instalación la consulta de libre/ocupado de un buzón de Exchange Online antes de volver a activar Graph API.

## Problemas conocidos

Ni el anuncio ni KB5133453 mencionan problemas conocidos para la SU de noviembre (a fecha de 11 de noviembre de 2026). El anuncio no tiene comentarios del equipo de Exchange y Microsoft indica que la documentación puede estar aún incompleta en el momento de la publicación. No queda ningún problema conocido abierto de los meses anteriores: los problemas de las actualizaciones de septiembre están resueltos desde la SU de octubre, y el problema de Windows de octubre se ha corregido con las actualizaciones de noviembre para Windows Server (véase arriba).

## Instalación y tareas posteriores

Microsoft describe en el anuncio el procedimiento habitual: realizar un inventario con [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) para determinar qué servidores necesitan actualizaciones o pasos manuales, instalar la SU, reiniciar el servidor y comprobar que todos los servicios de Exchange estén en ejecución. Si los servicios quedan desactivados tras la instalación, la instalación se interrumpió. En caso de errores durante o después de la instalación, Microsoft remite al [script SetupAssist](https://aka.ms/ExSetupAssist). Según KB5133453, tras la instalación Health Checker muestra si la SU se ha instalado correctamente y si se requieren pasos adicionales.

Según KB5133453, el archivo de instalación se llama `ExchangeSubscriptionEdition-KB5133453-x64-en.exe` (Download Center: «SU12»). La SU sustituye la SU de octubre KB5128895. Las SU son acumulativas: quien tenga instalada una SU anterior en Exchange SE RTM puede instalar directamente la SU de noviembre.

Según las preguntas frecuentes, para entornos híbridos se aplica lo siguiente: si se cambia el certificado de autenticación después de instalar una SU, debe ejecutarse de nuevo Hybrid Configuration Wizard. En Windows Server 2025, las SU de Exchange instaladas no aparecen en el Panel de control; Microsoft desaconseja desinstalar las SU.

De los meses anteriores, siguen pendientes estas tareas posteriores, si aún no se han realizado: revertir la invalidación de libre/ocupado `EnableRouteThroughMSGraphFeature` (véase el [artículo de octubre](/blog/exchange-security-updates-oktober-2026)), eliminar SettingOverride del contenedor `DisableBlockSharedAndUserMailboxHeaders` (véase el [artículo de septiembre](/blog/exchange-security-updates-september-2026)) y comprobar si la mitigación CVE-2026-42897 (M2.1.0) sigue activa (véase el [artículo sobre la SU de julio](/blog/exchange-security-updates-juli-2026)).

## Procedimiento recomendado

Instale pronto la SU de noviembre en todos los servidores con Exchange SE y en los equipos con Management Tools: para la ejecución remota de código crítica CVE-2026-118207 basta una cuenta con pocos privilegios y Microsoft considera más probable su explotación. Instale en la misma ventana de mantenimiento las actualizaciones de noviembre para Windows Server para corregir la causa de las interrupciones de octubre. Ya no hay parches para Exchange 2016 y 2019; según Microsoft, estos servidores deben considerarse permanentemente vulnerables y sustituirse inmediatamente por Exchange SE.

## Fuentes

1.  [Released: November 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-november-2026-exchange-server-security-updates/4570368): Anuncio del 10 de noviembre de 2026 con la versión compatible, nota sobre Exchange 2016/2019, el problema de Windows corregido y preguntas frecuentes (sin comentarios del equipo de Exchange, a fecha de 11 de noviembre de 2026).

2.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM November 10, 2026 (KB5133453) – Microsoft Support](https://support.microsoft.com/help/5133453): Lista de CVE, problema corregido, archivo de instalación, actualización sustituida y nota sobre Health Checker.

3.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Tipo, CVSS, gravedad, valoración de explotación, preguntas frecuentes y producto afectado de los nueve CVE de noviembre.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Número de compilación de la SU de noviembre.

5.  [Microsoft graph displays incorrect free/busy availability information in Microsoft Exchange server SE hybrid setup – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5135867): Síntomas y corrección con KB5133453.

6.  [Exchange servers may become unstable after installation of October 2026 Windows updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/exchange-servers-may-become-unstable-after-installation-of/ba-p/4565382): Publicación del equipo de Exchange sobre el problema de Windows de octubre, a la que remite el anuncio.

7.  [November 10, 2026: KB5131080 (OS Build 26100.33822) – Microsoft Support](https://support.microsoft.com/help/5131080): Notas de la versión de Windows Server 2025 con la corrección de Exchange y el problema conocido marcado como resuelto.

8.  [November 10, 2026: KB5131090 (OS Build 20348.5870) – Microsoft Support](https://support.microsoft.com/help/5131090): Notas de la versión de Windows Server 2022 con la misma corrección.

9.  [November 10, 2026: KB5131085 (OS Build 17763.9487) – Microsoft Support](https://support.microsoft.com/help/5131085): Notas de la versión de Windows Server 2019 con la misma corrección.

10. [November 10, 2026: KB5131291 (OS Build 14393.9688) – Microsoft Support](https://support.microsoft.com/help/5131291): Notas de la versión de Windows Server 2016 con la misma corrección.

11. [Nuevas actualizaciones de seguridad para Exchange Server (noviembre de 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-november-2026/): Desglose en alemán de los nueve CVE y contexto para Exchange 2016 y 2019.

12. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventario de versiones de CU/SU antes y después de la instalación.
