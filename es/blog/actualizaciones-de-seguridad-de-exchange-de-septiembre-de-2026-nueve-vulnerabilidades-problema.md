---
title: "Actualizaciones de seguridad de Exchange de septiembre de 2026: nueve vulnerabilidades, problema de los wrappers corregido y v2 publicada"
navTitle: "Exchange SU 09/2026"
description: "La SU de septiembre corrige nueve vulnerabilidades en Exchange SE y 2019 (ocho en Exchange 2016), incluida una vulnerabilidad de suplantación con CVSS 9.3, y soluciona el problema de los wrappers en entornos híbridos. El 2 de octubre se publicó una v2 con una CVE adicional; además, hay tres problemas conocidos con soluciones alternativas y un SettingOverride que ahora debe eliminarse."
date: "2026-10-07"
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
slug: "actualizaciones-de-seguridad-de-exchange-de-septiembre-de-2026-nueve-vulnerabilidades-problema"
translationId: article-53db0c02bc33f9bd
translationOf: exchange-security-updates-september-2026
url: https://rafaelpfister.ch/es/blog/actualizaciones-de-seguridad-de-exchange-de-septiembre-de-2026-nueve-vulnerabilidades-problema
translationSourceHash: d0738f61713a26973457a9e536720b9787af35b22867c648dbe113d2aeb3f4ea
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T10:45:43.442Z
translationReview: automatic
---

# Actualizaciones de seguridad de Exchange de septiembre de 2026: nueve vulnerabilidades, problema de los wrappers corregido y v2 publicada

Microsoft publicó el 8 de septiembre de 2026 actualizaciones de seguridad (SU) para Exchange Server. Corrigen nueve vulnerabilidades en Exchange SE y Exchange 2019, y ocho en Exchange 2016. Ninguna era conocida públicamente de antemano, ninguna está siendo explotada activamente según la Security Update Guide, y Microsoft las clasifica a todas como *Important* con «Exploitation Less Likely». Sin embargo, el valor CVSS más alto, 9.3, supera claramente al del mes anterior. Este mes es especial por tres motivos: la SU corrige el problema de los *mensajes wrapper* en buzones compartidos, abierto desde junio; introduce tres problemas conocidos nuevos o persistentes; y Microsoft publicó el 2 de octubre una **versión 2** que corrige una vulnerabilidad adicional.

## Para qué versiones de Exchange está disponible la actualización

Las SU del 8 de septiembre de 2026 están disponibles para las siguientes versiones:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121608, compilación 15.2.2562.49; disponible públicamente.
- **Exchange Server 2019 CU15**: KB5121609, compilación 15.2.1748.51; solo mediante el **programa ESU Period 2**.
- **Exchange Server 2019 CU14**: KB5121610, compilación 15.2.1544.46; solo mediante ESU Period 2.
- **Exchange Server 2016 CU23**: KB5121611, compilación 15.1.2507.73; solo mediante ESU Period 2.

Exchange 2016 y 2019 están fuera de soporte. Según Microsoft, las SU de mayo a octubre de 2026 solo las reciben las organizaciones inscritas en el programa ESU Period 2. Según los artículos KB, esta autorización se extiende hasta octubre de 2026. También existe presión desde Exchange Online: desde la segunda semana de septiembre, Exchange Online limita y bloquea el flujo de correo híbrido de servidores con versiones anteriores a octubre de 2025; consulte los detalles en el [artículo sobre la aplicación de transporte](/blog/exchange-online-transport-enforcement-hybrid-server). Según el anuncio, Exchange Online ya está protegido; no obstante, en entornos híbridos todos los servidores Exchange necesitan la SU, al igual que las máquinas con las Exchange Management Tools.

Puede comparar su propia versión con la lista de [números de compilación de Exchange](/tools/exchange-builds).

## Resumen de las vulnerabilidades

| CVE | Tipo | CVSS |
| --- | --- | --- |
| CVE-2026-69356 | Suplantación (Cross-Site Scripting) | 9.3 |
| CVE-2026-69641 | Elevation of Privilege | 9.1 |
| CVE-2026-69355 | Remote Code Execution | 8.8 |
| CVE-2026-55007 | Remote Code Execution | 8.1 |
| CVE-2026-69380 | Elevation of Privilege | 8.1 |
| CVE-2026-69378 | Denial of Service | 7.5 |
| CVE-2026-69361 | Suplantación (Server-Side Request Forgery) | 6.5 |
| CVE-2026-69375 | Tampering | 6.5 |
| CVE-2026-69382 | Information Disclosure | 5.9 |

CVE-2026-55007 no afecta a Exchange 2016; la Security Update Guide solo incluye Exchange SE y 2019 CU14/CU15. Un detalle de la documentación: CVE-2026-69380 no aparece en la lista de CVE de los artículos KB de las cuatro SU de septiembre. Sin embargo, la Security Update Guide indica precisamente las cuatro compilaciones de septiembre como corrección para esta CVE (a 7 de octubre de 2026).

**[CVE-2026-69356](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69356)** tiene el valor más alto, CVSS 9.3. Según Microsoft, un atacante no autenticado puede enviar una invitación de calendario manipulada con un enlace de reunión malicioso; si la destinataria o el destinatario abre la reunión y selecciona el enlace para unirse, se desencadena el Cross-Site Scripting. Por tanto, requiere interacción del usuario, pero no una cuenta en la organización.

**[CVE-2026-69380](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69380)** (Elevation of Privilege, CVSS 8.1) solo requiere una cuenta con pocos privilegios y un buzón asignado. Según las preguntas frecuentes de la Security Update Guide, un atacante puede aprovechar debilidades en la validación de solicitudes y tokens de identidad para hacerse pasar por otro usuario y tomar control de los buzones de todos los usuarios de Exchange: leer y enviar correos, y descargar adjuntos. Basta una única cuenta de usuario comprometida como punto de partida.

**[CVE-2026-69641](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69641)** (Elevation of Privilege, CVSS 9.1) conduce al mismo resultado, la toma de control de todos los buzones, pero requiere pertenecer a un grupo de roles con altos privilegios.

La situación inicial es diferente para las dos vulnerabilidades de ejecución remota de código: [CVE-2026-69355](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-69355) (CVSS 8.8) requiere una cuenta autenticada con pocos privilegios, mientras que [CVE-2026-55007](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-55007) (CVSS 8.1) puede desencadenarse sin iniciar sesión mediante un adjunto de Visio manipulado, aunque según Microsoft requiere memoria disponible persistentemente escasa en el sistema de destino. Las otras cuatro vulnerabilidades son: CVE-2026-69378 (DoS mediante recursión no controlada, sin inicio de sesión), CVE-2026-69361 (SSRF, el servidor envía solicitudes HTTP a sistemas internos o de loopback), CVE-2026-69375 (un atacante autenticado puede sustituir contenidos de archivos) y CVE-2026-69382 (divulgación de credenciales mediante un algoritmo criptográfico débil, requiere una cookie de autenticación obtenida previamente).

## Versión 2 del 2 de octubre: CVE-2026-96940 añadida

El 2 de octubre de 2026, Microsoft publicó la «versión 2» de las SU de septiembre. Según el anuncio, la única diferencia respecto a la primera versión es la corrección adicional de **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940). Esta vulnerabilidad de Elevation of Privilege tiene CVSS 8.8, no es conocida públicamente ni está siendo explotada, pero Microsoft la evalúa como la única de las diez CVE con **«Exploitation More Likely»**. Un atacante autenticado puede acceder a buzones ajenos de la misma organización y leer correos con sus adjuntos. Exchange Online ya está corregido en el lado del servidor.

| Versión | KB | Compilación v2 |
| --- | --- | --- |
| Exchange SE RTM | KB5129955 | 15.2.2562.53 |
| Exchange 2019 CU15 | KB5129956 | 15.2.1748.53 |
| Exchange 2019 CU14 | KB5129957 | 15.2.1544.48 |
| Exchange 2016 CU23 | KB5129958 | 15.1.2507.75 |

En la práctica, esto significa que la corrección de CVE-2026-96940 solo está incluida en las compilaciones v2. Los servidores que ya ejecutan la SU del 8 de septiembre necesitan instalar adicionalmente v2. Quienes apliquen el parche ahora pueden instalar directamente v2, ya que las SU son acumulativas. Los artículos KB no indican expresamente si los servidores con la primera SU de septiembre deben instalar obligatoriamente v2; dado que la nueva CVE solo se corrige con v2, esa es la interpretación lógica.

## Problema de los wrappers corregido: elimine ahora el SettingOverride

El problema conocido desde la SU de junio, por el que aparecen *mensajes wrapper* en la bandeja de entrada de buzones compartidos en entornos híbridos, se ha corregido con la SU de septiembre en las cuatro versiones. En el [artículo de agosto](/blog/exchange-security-updates-august-2026) todavía se indicaba que el SettingOverride documentado como solución alternativa podía mantenerse. Tras instalar la SU de septiembre, ocurre lo contrario: Microsoft recomienda en el artículo de soporte correspondiente comprobar y eliminar el override.

```powershell
Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Comando | Efecto |
|---|---|
| `Get-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Comprueba si el override de la solución alternativa está configurado en la organización. |
| `Remove-SettingOverride "DisableBlockSharedAndUserMailboxHeaders"` | Elimina el override una vez instalada la SU de septiembre. |

</details>

Si el primer comando informa de que no se encontró el objeto `DisableBlockSharedAndUserMailboxHeaders`, según Microsoft no hay nada más que hacer.

Para Exchange SE, la SU también corrige un error en las consultas híbridas de libre/ocupado mediante Microsoft Graph: los usuarios locales veían las horas de ocupación de los buzones de Exchange Online desplazadas por su propio desfase UTC, sin mensaje de error.

## Problemas conocidos

**Los calendarios publicados (.ics) devuelven HTTP 500 a las aplicaciones de calendario.** El problema existe desde la SU de agosto (Exchange SE a partir de la compilación 15.2.2562.46, así como Exchange 2019 y 2016) y tampoco se ha corregido en la SU de septiembre ni en v2. Las suscripciones a calendarios publicados anónimamente ya no se actualizan; la misma URL funciona en el navegador. Según Microsoft, la causa es que Exchange identifica a los clientes mediante el User-Agent; las aplicaciones de calendario sin identificador de navegador caen en una ruta de código que la SU de agosto desactivó. Como solución alternativa, Microsoft describe una regla de reescritura de URL en el sitio «Exchange Back End» de IIS que añade el parámetro `layout=premium` a las solicitudes .ics bajo `/owa/calendar/`; para ello debe estar instalado el módulo IIS URL Rewrite. Los pasos exactos, mediante IIS Manager o directamente en `applicationHost.config`, se encuentran en el [artículo de soporte KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672). Microsoft no indica una fecha para la corrección (a 7 de octubre de 2026).

**Libre/ocupado para buzones delegados en entornos híbridos (solo Exchange SE).** Si la consulta de disponibilidad está configurada exclusivamente mediante Graph API, las consultas de buzones de Exchange Online mediante acceso local delegado fallan. Outlook muestra «Your server location could not be determined», OWA muestra «No information» y en los registros EWS aparece `(403) Forbidden`. La solución alternativa documentada vuelve a dirigir las consultas mediante EWS en lugar de Graph:

```powershell
Set-SettingOverride -Identity EnableRouteThroughMSGraphFeature -Parameters "Enabled=False"
Get-ExchangeDiagnosticInfo -Process Microsoft.Exchange.Directory.TopologyService -Component VariantConfiguration -Argument Refresh
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Identity EnableRouteThroughMSGraphFeature` | El override que controla el reenvío de las consultas de disponibilidad mediante Microsoft Graph. |
| `-Parameters "Enabled=False"` | Desactiva la ruta de Graph; las consultas vuelven a realizarse mediante EWS. |
| `-Process Microsoft.Exchange.Directory.TopologyService` | Dirige la llamada de diagnóstico al servicio de topología. |
| `-Component VariantConfiguration -Argument Refresh` | Recarga Variant Configuration para que el override surta efecto sin esperar. |

</details>

Microsoft incluye este problema entre los problemas corregidos en el anuncio de v2. Quienes hayan configurado la solución alternativa deberían comprobar, tras instalar v2, en el [artículo de soporte KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) si debe revertirse; a 7 de octubre de 2026 todavía no había instrucciones al respecto.

**Bloqueo mutuo de ContentEngine debido a archivos de reglas WordBreaker coreanos ausentes (solo Exchange SE).** Con la SU de septiembre (compilación 15.2.2562.49 y compilación v2 15.2.2562.53) no se instalan los archivos de reglas del WordBreaker coreano actualizado. Las consecuencias son resultados de búsqueda ausentes, entrega de correo retrasada y clientes Outlook o MAPI que se bloquean o pierden la conexión. Según el anuncio, afecta a mensajes en coreano. La solución alternativa del [artículo de soporte KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098) consiste en extraer los dos archivos `ko.token.rule.bin` y `ko.complex.rule.bin` de SQL Server 2025 Express RTM, comprobar sus hashes SHA256, copiarlos al directorio `Native` de la instalación de Exchange y reiniciar el servicio Search Host Controller. Microsoft sigue investigando el problema.

## Instalación y tareas posteriores

Microsoft recomienda el procedimiento habitual: inventariar con el [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), determinar la ruta con el [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) si la versión está desactualizada, instalar la SU, reiniciar el servidor y comprobar que todos los servicios de Exchange estén en ejecución. Después, Health Checker también muestra si la SU se ha instalado correctamente. La Security Update Guide indica que estas actualizaciones requieren reinicio.

Después de la instalación, hay tres tareas pendientes:

1. Elimine el Wrapper-SettingOverride `DisableBlockSharedAndUserMailboxHeaders` si está configurado (véase arriba).

2. En Exchange SE, compruebe si aparecen los problemas de libre/ocupado y WordBreaker, y aplique las soluciones alternativas si procede.

3. Para calendarios publicados con suscriptores externos, configure la regla de reescritura de URL de KB5126672 si no se hizo ya desde la SU de agosto.

Además, desde julio sigue siendo necesario comprobar si la mitigación CVE-2026-42897 (M2.1.0) continúa activa; cómo eliminarla se explica en el [artículo sobre la SU de julio](/blog/exchange-security-updates-juli-2026).

## Procedimiento recomendado

Instale directamente las compilaciones v2 del 2 de octubre en todos los servidores Exchange y las máquinas con Management Tools; los servidores con la SU del 8 de septiembre necesitan v2 adicionalmente para CVE-2026-96940. La vulnerabilidad de suplantación con CVSS 9.3 y la toma de control de buzones mediante una cuenta de usuario con pocos privilegios (CVE-2026-69380) son motivos suficientes para no esperar al próximo Patch Tuesday. Después, elimine el override de wrappers, compruebe los tres problemas conocidos y ejecute Health Checker. Para Exchange 2016 y 2019, el programa ESU finaliza en octubre de 2026; ya no se puede seguir posponiendo la migración a Exchange SE.

## Fuentes

1.  [Released: September 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-september-2026-exchange-server-security-updates/4554411): Anuncio oficial de la versión con las versiones compatibles, nota sobre ESU, problemas conocidos, problemas corregidos y procedimiento de instalación (consultado mediante el feed RSS del Exchange Team Blog).

2.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Anuncio de la v2 del 2 de octubre de 2026; la única diferencia es CVE-2026-96940.

3.  [Description of the security update for Microsoft Exchange Server Subscription Edition RTM: September 8, 2026 (KB5121608) – Microsoft Support](https://support.microsoft.com/help/5121608): Lista de CVE, problemas corregidos y los tres problemas conocidos para Exchange SE.

4.  [Description of the security update for Microsoft Exchange Server 2019 CU15: September 8, 2026 (KB5121609) – Microsoft Support](https://support.microsoft.com/help/5121609): Artículo KB para Exchange 2019 CU15.

5.  [Description of the security update for Microsoft Exchange Server 2019 CU14: September 8, 2026 (KB5121610) – Microsoft Support](https://support.microsoft.com/help/5121610): Artículo KB para Exchange 2019 CU14.

6.  [Description of the security update for Microsoft Exchange Server 2016 CU23: September 8, 2026 (KB5121611) – Microsoft Support](https://support.microsoft.com/help/5121611): Artículo KB para Exchange 2016 CU23, sin CVE-2026-55007.

7.  [Security Update Guide – Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/): Tipo, CVSS, gravedad, evaluación de explotación y preguntas frecuentes sobre las nueve CVE de septiembre y CVE-2026-96940, incluidas las compilaciones afectadas para cada CVE.

8.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): Artículo KB sobre la v2 para Exchange SE; los artículos v2 para 2019 y 2016 son KB5129956 a KB5129958.

9.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Números de compilación de las SU de septiembre y de la v2 del 2 de octubre de 2026.

10. [Wrapper messages appear in shared mailbox in hybrid environments after installing the June 2026 Security Update – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): Corrección en la SU de septiembre e instrucciones para eliminar el SettingOverride.

11. [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Causa y solución alternativa mediante reescritura de URL.

12. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Síntomas y solución alternativa mediante SettingOverride para Exchange SE.

13. [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Compilaciones SE afectadas y solución alternativa manual.

14. [Hybrid free/busy through Microsoft Graph incorrectly shifts busy times by requester timezone – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5125804): Error de zona horaria en Exchange SE corregido con la SU de septiembre.

15. [Neue Sicherheitsupdates für Exchange Server (September 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-september-2026/): Desglose en alemán de las nueve CVE con valores CVSS y compilaciones.

16. [Exchange Server: Sicherheitsupdates 8. September 2026 – Borns Tech and Windows World](https://borncity.com/blog/?p=329371): Resumen en alemán con referencias a la posterior actualización de reemplazo.
