---
title: "Actualizaciones de seguridad de Exchange de octubre de 2026: seis vulnerabilidades y última actualización para Exchange 2016 y 2019"
navTitle: "Exchange SU 10/2026"
description: "La SU de octubre corrige seis vulnerabilidades, incluida una ejecución remota de código crítica mediante un correo electrónico manipulado, y resuelve el error de .ics y el bloqueo de WordBreaker. Para Exchange 2016 y 2019 es la última actualización, también en el programa ESU."
date: "2026-10-14"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min de lectura"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
  - "powershell"
slug: "actualizaciones-de-seguridad-de-exchange-de-octubre-de-2026-seis-vulnerabilidades-ultima"
translationId: article-173db9431239a95f
translationOf: exchange-security-updates-oktober-2026
translationSourceHash: a834949e09b3f4b6e7f16be2dd6cf0ecedd0ccf1f4fc8dd6f409336407aaaca2
translationModel: gpt-5.6-terra
translatedAt: 2026-10-16T08:25:16.117Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/actualizaciones-de-seguridad-de-exchange-de-octubre-de-2026-seis-vulnerabilidades-ultima
---

# Actualizaciones de seguridad de Exchange de octubre de 2026: seis vulnerabilidades y última actualización para Exchange 2016 y 2019

Microsoft publicó el 13 de octubre de 2026 actualizaciones de seguridad (SU) para Exchange Server SE, Exchange 2019 y Exchange 2016. Según los artículos de KB, corrigen seis vulnerabilidades. Ninguna era conocida públicamente de antemano y, según la Security Update Guide, ninguna se está explotando activamente. Sin embargo, Microsoft clasifica dos como «Exploitation More Likely», incluida una ejecución remota de código calificada como *Critical* que un atacante no autenticado puede desencadenar con un correo electrónico manipulado. Este mes es especial por dos motivos: la SU resuelve los dos problemas conocidos pendientes de las actualizaciones de septiembre (HTTP 500 en calendarios publicados y el bloqueo de ContentEngine en Exchange SE), y para Exchange 2016 y 2019 es, según Microsoft, la última actualización de seguridad, también para los clientes del programa Period 2 ESU.

## Para qué versiones de Exchange está disponible la actualización

Las SU del 13 de octubre de 2026 están disponibles para las siguientes versiones:

- **Exchange Server Subscription Edition (SE) RTM**: KB5128895, compilación 15.2.2562.56; disponible públicamente.
- **Exchange Server 2019 CU15**: KB5128896, compilación 15.2.1748.55; solo mediante el **programa Period 2 ESU**.
- **Exchange Server 2019 CU14**: KB5128897, compilación 15.2.1544.50; solo mediante Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5128898, compilación 15.1.2507.77; solo mediante Period 2 ESU.

Puede comparar su versión actual con la lista de [números de compilación de Exchange](/tools/exchange-builds).

Exchange 2016 y 2019 están fuera de soporte. Según los artículos de KB, las organizaciones incluidas en el programa Period 2 ESU reciben actualizaciones de seguridad hasta finales de octubre de 2026; todas las demás deben migrar a Exchange SE. Según el anuncio, Exchange Online ya está protegido; aun así, en entornos híbridos cada servidor de Exchange necesita la SU, incluidos los servidores exclusivamente de administración y las máquinas con las Exchange Management Tools. Lo que significa concretamente el final de ESU se explica en la sección [Última actualización para Exchange 2016 y 2019](#letztes-update-für-exchange-2016-und-2019).

## Resumen de las vulnerabilidades

| CVE | Tipo | CVSS |
| --- | --- | --- |
| CVE-2026-70279 | Suplantación de identidad (Cross-Site-Scripting) | 9.3 |
| CVE-2026-77910 | Elevación de privilegios | 8.8 |
| CVE-2026-70571 | Ejecución remota de código | 8.1 |
| CVE-2026-87692 | Elevación de privilegios | 7.8 |
| CVE-2026-87695 | Elevación de privilegios | 7.8 |
| CVE-2026-87696 | Elevación de privilegios | 7.5 |

Los cuatro artículos de KB enumeran las mismas seis CVE. Sin embargo, en la Security Update Guide falta Exchange 2019 CU15 para CVE-2026-87692; allí solo figuran Exchange SE, 2019 CU14 y 2016 CU23 como afectados (a 14 de octubre de 2026). Frankys Web interpreta que 2019 CU15 no está afectado por esta vulnerabilidad. Para la instalación, la diferencia no importa, ya que la SU para CU15 corrige de todos modos todas las demás vulnerabilidades.

**[CVE-2026-70571](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-70571)** es la única vulnerabilidad del mes que Microsoft clasifica como *Critical* (ejecución remota de código, CVSS 8.1). La causa es un desbordamiento de búfer basado en pila. Según las preguntas frecuentes de la Security Update Guide, un atacante no autenticado puede enviar un correo electrónico manipulado a un servidor de Exchange afectado y desencadenar así un proceso de conversión interno; no se requiere interacción del usuario. Según el vector CVSS, la complejidad del ataque es alta (AC:H); Microsoft indica que no se garantiza una explotación satisfactoria y que requiere un conocimiento profundo del entorno objetivo. Aun así, Microsoft valora la explotación como «Exploitation More Likely». El atacante no necesita una cuenta en la organización; basta con que el servidor acepte correos.

**[CVE-2026-70279](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-70279)** tiene el valor más alto, con CVSS 9.3, y también está clasificada como «Exploitation More Likely». Se trata de un Cross-Site-Scripting: un atacante no autenticado envía un enlace manipulado; si un administrador lo abre, el atacante puede ejecutar acciones con sus permisos y, de este modo, acceder a los datos de la organización o modificarlos. Según las preguntas frecuentes, esto requiere interacción del usuario, normalmente mediante ingeniería social. La gravedad es *Important*.

**[CVE-2026-77910](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-77910)** (elevación de privilegios, CVSS 8.8) solo requiere una cuenta con pocos permisos. Según las preguntas frecuentes, un atacante puede manipular una caché compartida del Exchange Control Panel y hacer que se instale un complemento de correo malicioso en el buzón de otro usuario, sin que este tenga que hacer nada. Microsoft describe la misma vía de ataque para [CVE-2026-87696](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87696) (CVSS 7.5), en este caso como consecuencia de una comprobación insuficiente de firmas criptográficas y con alta complejidad de ataque.

Las dos vulnerabilidades restantes, [CVE-2026-87692](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87692) (deserialización de datos no confiables) y [CVE-2026-87695](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-87695), son elevaciones locales de privilegios con CVSS 7.8: según Microsoft, un atacante autenticado con pocos permisos puede obtener privilegios SYSTEM. Ambas están clasificadas como «Exploitation Less Likely», al igual que CVE-2026-77910 y CVE-2026-87696.

Sobre **CVE-2026-96940** de la v2 de septiembre: el anuncio de octubre no menciona la v2. Según las preguntas frecuentes del anuncio, las SU son acumulativas; quien esté en una CU compatible solo instala la SU más reciente. Frankys Web confirma que la SU de octubre incluye la corrección de la v2. Por tanto, los servidores que aún ejecutan la primera SU de septiembre, del 8 de septiembre, pueden instalar directamente la SU de octubre; los detalles sobre la v2 están en el [artículo sobre la v2 de septiembre](/blog/exchange-security-updates-september-2026-v2).

## Última actualización para Exchange 2016 y 2019

El anuncio es claro en este punto: octubre de 2026 es el último mes en el que los clientes del programa Period 2 ESU reciben nuevas actualizaciones de seguridad para Exchange 2016 y 2019. A partir de la siguiente versión, las actualizaciones solo se publicarán para Exchange SE. Los futuros anuncios y avisos de CVE dejarán de mencionar 2016 y 2019, y Microsoft escribe expresamente: «We will stop testing Exchange 2016 and 2019 for vulnerabilities.» Por tanto, que una CVE no mencione estas versiones no significa que no estén afectadas; Microsoft recalca en el anuncio que las versiones antiguas no se vuelven seguras por ello. Frankys Web también señala que Exchange SE y 2019 CU15 son técnicamente muy similares, por lo que futuras vulnerabilidades de SE probablemente afectarán también a 2019.

Según el anuncio, Period 2 ESU aún se puede comprar durante un tiempo limitado. Sin embargo, eso solo lleva a una organización al nivel de la actualización de octubre de 2026; no habrá más actualizaciones. Según los artículos de KB, quien ya haya adquirido una ESU y tenga preguntas sobre el acceso puede contactar con `ExchangeandSfBServerESUInquiry@service.microsoft.com`.

Para la operación, esto significa:

1. **Exchange 2016 y 2019 con ESU**: instale la SU de octubre. Después, la versión queda congelada y todas las vulnerabilidades que se conozcan posteriormente quedarán sin corregir.

2. **Exchange 2016 y 2019 sin ESU**: estos servidores no reciben actualizaciones de seguridad desde mayo de 2026. Microsoft recomienda migrar a Exchange SE lo antes posible.

3. **Entornos híbridos**: además de la falta de parches, Exchange Online limita y bloquea el flujo de correo procedente de servidores locales obsoletos; las reglas se detallan en el [artículo sobre la aplicación de reglas de transporte](/blog/exchange-online-transport-enforcement-hybrid-server).

## Problemas corregidos y qué debe hacer después

**Los calendarios publicados (.ics) devuelven HTTP 500 a aplicaciones de calendario** (todas las versiones). El problema existente desde la SU de agosto queda resuelto con la SU de octubre; [KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672) ahora indica como solución la instalación de las cuatro actualizaciones de octubre o de versiones posteriores. El artículo de soporte ya no describe la solución temporal anterior de reescritura de URL en el sitio «Exchange Back End», y Microsoft no proporciona instrucciones para eliminar la regla (a 14 de octubre de 2026). Si configuró la regla, documente en qué servidores está activa y pruebe una suscripción de calendario antes de eliminarla.

**Bloqueo de ContentEngine debido a archivos de reglas WordBreaker coreanos ausentes** (solo Exchange SE). Según [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098), el problema se corrige con KB5128895 o una actualización posterior y, tras instalar la actualización de octubre, no es necesaria «no additional action». Esto también se aplica a los servidores en los que aplicó la solución manual del [artículo de septiembre](/blog/exchange-security-updates-september-2026#workaround-für-den-wordbreaker-deadlock): Microsoft no exige revertir los archivos de reglas copiados. Los servidores sin solución temporal ya no necesitan añadir los archivos manualmente.

**Libre/ocupado para buzones delegados en entornos híbridos con configuración exclusiva de Graph API** (solo Exchange SE). La SU de octubre no corrige este problema: [KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) informa ahora: «The issue has been fixed in the service». Con ello queda resuelta la contradicción del [artículo sobre la v2](/blog/exchange-security-updates-september-2026-v2). Si estableció la solución temporal, reviértala según KB5127092:

```powershell
Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=True"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Get-SettingOverride -Identity EnableRouteThroughMSGraphFeature` | Muestra el valor actual de la invalidación; solo con `Enabled=False` es necesario el siguiente comando. |
| `-Parameters "Enabled=True"` | Vuelve a activar el reenvío de consultas de disponibilidad mediante Microsoft Graph. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Dirige la llamada de diagnóstico al servicio de topología. |
| `-Component VariantConfiguration -Argument Refresh` | Recarga la Variant Configuration para que el cambio tenga efecto inmediatamente. |

</details>

Después, compruebe la consulta de libre/ocupado de un buzón delegado de Exchange Online desde Outlook u OWA.

## Problemas conocidos

Ni el anuncio ni los cuatro artículos de KB mencionan problemas conocidos para la SU de octubre (a 14 de octubre de 2026). El 15 de octubre de 2026, Microsoft complementó el anuncio: las actualizaciones de Windows Server de octubre de 2026 pueden provocar que los servidores de Exchange fallen tras aproximadamente un día con `MapiExceptionMdbOffline` (ec=1142), incluso sin tener instalada una SU de Exchange. Los síntomas y el procedimiento se describen en el [artículo sobre las interrupciones de Exchange tras las actualizaciones de Windows](/blog/exchange-windows-updates-oktober-2026). Con ello quedan resueltos los tres problemas pendientes de las actualizaciones de septiembre: dos mediante la SU de octubre y uno mediante una corrección en Exchange Online (véase arriba). El problema del wrapper en buzones compartidos ya se había corregido con la SU de septiembre.

## Instalación y tareas posteriores

Microsoft indica en el anuncio el procedimiento habitual: inventariar con el [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), determinar la ruta con el [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) si la CU está obsoleta, instalar la SU, reiniciar el servidor y comprobar que se ejecuten todos los servicios de Exchange. En caso de errores, Microsoft remite al [script SetupAssist](https://aka.ms/ExSetupAssist). Según KB5128895, tras la instalación el Health Checker indica si la SU se instaló correctamente y si se requieren pasos adicionales. La Security Update Guide no indica un reinicio obligatorio para las compilaciones de octubre; aun así, el anuncio lo recomienda después de la configuración.

Según KB5128895, el archivo de instalación para Exchange SE se llama `ExchangeSubscriptionEdition-KB5128895-x64-en.exe` (Download Center: «SU11»), para Exchange 2019 CU15 `Exchange2019-KB5128896-x64-en.exe`, para CU14 `Exchange2019-KB5128897-x64-en.exe` y para Exchange 2016 `Exchange2016-KB5128898-x64-en.exe`. Los paquetes son específicos de cada CU.

Según las preguntas frecuentes, para entornos híbridos se aplica lo siguiente: si se cambia el certificado de autenticación tras instalar una SU, debe ejecutarse de nuevo el Hybrid Configuration Wizard. En Windows Server 2025, las SU de Exchange instaladas no aparecen en el Panel de control; Microsoft desaconseja desinstalar las SU.

De los meses anteriores quedan pendientes estas tareas, si aún no se han realizado: eliminar la SettingOverride del wrapper `DisableBlockSharedAndUserMailboxHeaders` (véase el [artículo de septiembre](/blog/exchange-security-updates-september-2026)) y comprobar si la mitigación de CVE-2026-42897 (M2.1.0) sigue activa (véase el [artículo sobre la SU de julio](/blog/exchange-security-updates-juli-2026)).

## Procedimiento recomendado

Instale cuanto antes la SU de octubre en todos los servidores de Exchange y las máquinas con Management Tools: la ejecución remota de código crítica CVE-2026-70571 puede desencadenarse mediante un correo electrónico sin necesidad de una cuenta, y Microsoft considera más probable su explotación, al igual que la del Cross-Site-Scripting CVE-2026-70279. Con ello, los servidores con la primera SU de septiembre también reciben la corrección para CVE-2026-96940. Después, revierta la invalidación de libre/ocupado si se estableció y compruebe los calendarios publicados. Para Exchange 2016 y 2019, esta es la última actualización; planifique ahora la migración a Exchange SE, ya que a partir de noviembre estos servidores no recibirán más actualizaciones de seguridad.

## Fuentes

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-october-2026-exchange-server-security-updates/ba-p/4563014): Anuncio del 13 de octubre de 2026 con las versiones compatibles, los problemas corregidos, el final de las actualizaciones para Exchange 2016/2019 y las preguntas frecuentes (consultado mediante el canal RSS del Exchange Team Blog; sin comentarios del equipo de Exchange, a 14 de octubre de 2026).

2.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM October 13, 2026 (KB5128895) – Microsoft Support](https://support.microsoft.com/help/5128895): Lista de CVE, problemas corregidos, archivo de instalación y referencia al Health Checker.

3.  [Description of the security update for Microsoft Exchange Server 2019 CU15 October 13, 2026 (KB5128896) – Microsoft Support](https://support.microsoft.com/help/5128896): Artículo de KB para Exchange 2019 CU15 con indicación de ESU.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU14 October 13, 2026 (KB5128897) – Microsoft Support](https://support.microsoft.com/help/5128897): Artículo de KB para Exchange 2019 CU14 con dirección de contacto de ESU.

5.  [Description of the security update for Microsoft Exchange Server 2016 CU23 October 13, 2026 (KB5128898) – Microsoft Support](https://support.microsoft.com/help/5128898): Artículo de KB para Exchange 2016 CU23.

6.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Tipo, CVSS, gravedad, evaluación de explotación, preguntas frecuentes y compilaciones afectadas de las seis CVE de octubre, así como de CVE-2026-96940.

7.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Números de compilación de las SU de octubre y de las actualizaciones anteriores.

8.  [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Corrección con las actualizaciones de octubre.

9.  [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Corrección con KB5128895; no se requieren pasos adicionales.

10. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Corrección en el servicio y comandos para restablecer la SettingOverride.

11. [Nuevas actualizaciones de seguridad para Exchange Server (octubre de 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-oktober-2026/): Desglose en alemán de las seis CVE, indicación de la corrección v2 incluida y clasificación del final de ESU.

12. [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventario de los niveles de CU/SU antes y después de la instalación.
