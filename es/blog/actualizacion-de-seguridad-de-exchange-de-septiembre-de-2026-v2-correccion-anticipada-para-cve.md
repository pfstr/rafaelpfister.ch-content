---
title: "Actualización de seguridad de Exchange de septiembre de 2026 v2: corrección anticipada para CVE-2026-96940"
navTitle: "Exchange SU 09/2026 v2"
description: "El 2 de octubre de 2026, Microsoft publicó una versión 2 de las SU de septiembre para Exchange SE, 2019 y 2016. También corrige CVE-2026-96940, una vulnerabilidad de elevación de privilegios con CVSS 8.8 y la evaluación «Exploitation More Likely». Microsoft recomienda instalar v2 lo antes posible, también en servidores con la primera SU de septiembre."
date: "2026-10-07"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "5 min de lectura"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
protokolle:
  - "releases"
slug: "actualizacion-de-seguridad-de-exchange-de-septiembre-de-2026-v2-correccion-anticipada-para-cve"
translationId: article-3ae7bbe7175754d9
translationOf: exchange-security-updates-september-2026-v2
translationSourceHash: da0e0876babdd9cbb62e51789fe45b0dae627891d40970ac10590f35289018c8
translationModel: gpt-5.6-terra
translatedAt: 2026-10-25T19:44:16.377Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/actualizacion-de-seguridad-de-exchange-de-septiembre-de-2026-v2-correccion-anticipada-para-cve
---

# Actualización de seguridad de Exchange de septiembre de 2026 v2: corrección anticipada para CVE-2026-96940

El 2 de octubre de 2026, Microsoft publicó una **versión 2 (v2)** de las actualizaciones de seguridad (SU) de septiembre para Exchange Server SE, Exchange 2019 y Exchange 2016. Según el anuncio, la única diferencia respecto a la primera versión del 8 de septiembre es que v2 también corrige **CVE-2026-96940**. La vulnerabilidad no es públicamente conocida ni está siendo explotada; Microsoft la encontró internamente. Hay dos aspectos inusuales: según Microsoft, esta actualización se publicó antes de lo previsto, y la Security Update Guide clasifica la explotación como **«Exploitation More Likely»**. Las nueve vulnerabilidades de la primera versión, el problema corregido del wrapper y los problemas conocidos se describen en el [artículo sobre la SU de septiembre](/blog/exchange-security-updates-september-2026); este artículo trata sobre las novedades de v2. Desde el 13 de octubre de 2026 está disponible la [SU de octubre](/blog/exchange-security-updates-oktober-2026); dado que, según Microsoft, las SU son acumulativas, también contiene la corrección para CVE-2026-96940.

## Para qué versiones de Exchange está disponible la actualización

Las SU v2 del 2 de octubre de 2026 están disponibles para las siguientes versiones:

- **Exchange Server Subscription Edition (SE) RTM**: KB5129955, compilación 15.2.2562.53; disponible públicamente.
- **Exchange Server 2019 CU15**: KB5129956, compilación 15.2.1748.53; solo mediante el **programa ESU de período 2**.
- **Exchange Server 2019 CU14**: KB5129957, compilación 15.2.1544.48; solo mediante ESU de período 2.
- **Exchange Server 2016 CU23**: KB5129958, compilación 15.1.2507.75; solo mediante ESU de período 2.

Las actualizaciones son específicas de cada CU: el paquete para CU15 no se puede instalar en CU14. Exchange 2016 y 2019 están fuera de soporte. Según las preguntas frecuentes del anuncio, solo las organizaciones incluidas en el programa ESU de período 2, que va de mayo a octubre de 2026, reciben actualizaciones publicadas después de mayo de 2026 para estas versiones; Microsoft recomienda a todos los demás migrar a Exchange SE lo antes posible. Para entornos híbridos con servidores obsoletos, se añade la [aplicación de transporte de Exchange Online](/blog/exchange-online-transport-enforcement-hybrid-server).

Más recientes que v2 son las SU de octubre del 13 de octubre de 2026: KB5128895 (SE, compilación 15.2.2562.56), KB5128896 (2019 CU15, 15.2.1748.55), KB5128897 (2019 CU14, 15.2.1544.50) y KB5128898 (2016 CU23, 15.1.2507.77). Según el anuncio de octubre, para Exchange 2016 y 2019 se trata de la última actualización de seguridad, también en el programa ESU de período 2; posteriormente, las actualizaciones solo se publicarán para Exchange SE.

Puede comprobar si sus servidores ya están en el nivel v2 mediante el resumen de [números de compilación de Exchange](/tools/exchange-builds).

## Resumen de la vulnerabilidad

| CVE | Tipo | CVSS |
| --- | --- | --- |
| CVE-2026-96940 | Elevación de privilegios | 8.8 |

Microsoft describe **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940) como una autorización débil en Exchange Server mediante la cual un atacante autenticado puede ampliar sus privilegios a través de la red. El vector CVSS indica requisitos bajos: ataque a través de la red, baja complejidad, una cuenta con pocos privilegios y ninguna interacción del usuario. Según las preguntas frecuentes de la Security Update Guide, un atacante con éxito obtiene acceso no autorizado a los buzones de otros usuarios de la misma organización y puede leer sus correos electrónicos, incluidos los adjuntos; el acceso se limita a su propia organización. Microsoft evalúa la vulnerabilidad como *Important* y la puntuación temporal es de 7.7.

La evaluación «Exploitation More Likely» es la diferencia más importante frente a las nueve CVE de la primera versión, que Microsoft evaluó todas como «Exploitation Less Likely». El punto de partida de un ataque es cualquier cuenta de usuario válida, por ejemplo tras un phishing exitoso. Microsoft corrigió Exchange Online en el lado del servidor; allí no hay que hacer nada. Microsoft no indica ninguna mitigación ni solución alternativa para servidores sin v2 (a fecha del 7 de octubre de 2026).

Las demás CVE de los artículos KB de v2 son las mismas que en la primera versión: CVE-2026-55007, CVE-2026-69355, CVE-2026-69356, CVE-2026-69361, CVE-2026-69375, CVE-2026-69378, CVE-2026-69382 y CVE-2026-69641. Al igual que en los KB de septiembre, falta CVE-2026-69380, que según la Security Update Guide también se corrige con la SU de septiembre.

## Por qué existe una v2 y qué debe hacer

En las preguntas frecuentes del anuncio, Microsoft responde así a la pregunta sobre la secuencia inusual: la actualización con CVE-2026-96940 se publicó antes de la fecha prevista, y Microsoft recomienda revisar las indicaciones de implementación e instalar la actualización en la primera oportunidad posible. Microsoft no indica por qué se adelantó la fecha.

De ello se desprende lo siguiente para la práctica:

1. **Los servidores con la SU del 8 de septiembre** (compilaciones 15.2.2562.49, 15.2.1748.51, 15.2.1544.46, 15.1.2507.73) no están protegidos contra CVE-2026-96940. La Security Update Guide enumera exclusivamente las cuatro compilaciones v2 como corrección para esta CVE. Estos servidores necesitan v2 o, desde el 13 de octubre de 2026, la SU de octubre.

2. **Los servidores con una versión anterior** pueden instalar directamente la SU más reciente, actualmente la SU de octubre. Según las preguntas frecuentes, las SU son acumulativas; quien esté en una CU compatible solo instala la SU más reciente.

3. **Las máquinas con las Exchange Management Tools** y los servidores de administración exclusivos también reciben v2; Microsoft lo recomienda para todas las SU, también en entornos híbridos.

Según KB5129955, el archivo de instalación para Exchange SE se llama `ExchangeSubscriptionEdition-KB5129955-x64-en.exe`; está disponible a través de Microsoft Update Catalog y Download Center, donde aparece con la denominación «SU10V2».

## Problemas conocidos

Los problemas conocidos de la primera versión también se aplican a v2. Para los tres ya existe una solución (a fecha del 25 de octubre de 2026):

**Los calendarios publicados (.ics) devuelven HTTP 500 a las aplicaciones de calendario** (todas las versiones). El problema existe desde la SU de agosto y sigue figurando como problema conocido en los cuatro KB de v2. v2 no lo corrige; según el [artículo de soporte KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672), se corrige con las SU de octubre del 13 de octubre de 2026 (KB5128895 a KB5128898) o con actualizaciones posteriores. Si como solución alternativa configuró la regla de reescritura de URL en el sitio «Exchange Back End» (véase el [artículo sobre la SU de septiembre](/blog/exchange-security-updates-september-2026)), elimínela según KB5126672 después de instalar la actualización mediante los pasos de reversión descritos allí; la regla puede estar en applicationHost.config o en web.config de Exchange.

**Interbloqueo de ContentEngine debido a la falta de archivos WordBreaker coreanos** (solo Exchange SE). Según [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098), ambas versiones están expresamente afectadas: la compilación 15.2.2562.49 y la compilación v2 15.2.2562.53. Por tanto, v2 tampoco corrige el problema. Los síntomas son resultados de búsqueda ausentes, entrega de correo retrasada y clientes Outlook o MAPI bloqueados o desconectados. Según el KB, se ven afectados los entornos que procesan correos con contenido en coreano; no depende del idioma de los propios usuarios, ya que el texto coreano también llega a organizaciones sin correspondencia coreana, por ejemplo como spam. Health Checker no detecta los archivos que faltan. Según KB5130098, el problema se corrige con la SU de octubre KB5128895 del 13 de octubre de 2026 o con una actualización posterior; quien haya aplicado la solución alternativa manual del [artículo sobre la SU de septiembre](/blog/exchange-security-updates-september-2026#workaround-für-den-wordbreaker-deadlock) no tiene que hacer nada más después, según Microsoft.

**Libre/ocupado para buzones delegados en entornos híbridos con configuración exclusiva de Graph API** (solo Exchange SE). Cuando se publicó v2, las fuentes se contradecían: el anuncio de v2 incluía el problema entre los problemas corregidos, mientras que KB5129955 lo indicaba como problema conocido. Entretanto, el [artículo de soporte KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) informa: «The issue has been fixed in the service». Si desactivó el SettingOverride `EnableRouteThroughMSGraphFeature`, vuelva a establecerlo en `Enabled=True` según KB5127092; los comandos se encuentran en el [artículo sobre la SU de octubre](/blog/exchange-security-updates-oktober-2026).

## Instalación y tareas posteriores

Microsoft indica en el anuncio el procedimiento habitual: inventariar con [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), determinar la ruta con [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) si el nivel de CU está desactualizado, reiniciar el servidor después de la instalación y comprobar que todos los servicios de Exchange se hayan iniciado. En caso de errores durante o después de la instalación, Microsoft remite al [script SetupAssist](https://aka.ms/ExSetupAssist). Para las compilaciones v2, la Security Update Guide no indica un reinicio obligatorio para CVE-2026-96940; no obstante, el anuncio recomienda reiniciar después de la instalación.

Para los entornos híbridos, según las preguntas frecuentes, se aplica lo siguiente: si después de instalar una SU se cambia el certificado de autenticación, debe ejecutarse de nuevo Hybrid Configuration Wizard. En Windows Server 2025, las SU de Exchange instaladas no aparecen en el Panel de control; Microsoft desaconseja en general desinstalar las SU y remite, para este caso, a un artículo de soporte específico.

De los meses anteriores quedan pendientes estas tareas posteriores, si aún no se han realizado: eliminar el wrapper-SettingOverride `DisableBlockSharedAndUserMailboxHeaders` (véase el [artículo de septiembre](/blog/exchange-security-updates-september-2026)) y comprobar si la mitigación de CVE-2026-42897 (M2.1.0) sigue activa (véase el [artículo sobre la SU de julio](/blog/exchange-security-updates-juli-2026)).

## Procedimiento recomendado

Instale pronto v2 o la SU de octubre más reciente en todos los servidores Exchange y máquinas con Management Tools, incluso donde ya se ejecuta la SU de septiembre: la primera SU de septiembre no corrige CVE-2026-96940, una sola cuenta de usuario comprometida basta para acceder a buzones ajenos y Microsoft considera más probable la explotación que para todas las demás CVE de septiembre. A continuación, compruebe el nivel de compilación con Health Checker y, después de la SU de octubre, revierta las soluciones alternativas para los problemas conocidos como se describe arriba. Para Exchange 2016 y 2019, la SU de octubre del 13 de octubre de 2026 es la última actualización; después ya no se publicarán actualizaciones de seguridad para estas versiones.

## Fuentes

1.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Anuncio de v2 del 2 de octubre de 2026 con la diferencia respecto a la primera versión, las preguntas frecuentes sobre la publicación anticipada, problemas conocidos, problemas corregidos y procedimiento de instalación (consultado mediante el feed RSS del Exchange Team Blog).

2.  [CVE-2026-96940 – Security Update Guide, Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940): Tipo, CVSS 8.8/7.7 con vector, gravedad, evaluación de explotación, preguntas frecuentes y las cuatro compilaciones v2 como corrección.

3.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): Lista de CVE, problemas conocidos, problemas corregidos y archivo de instalación para Exchange SE.

4.  [Description of version 2 of the security update for Microsoft Exchange Server 2019 CU15 (KB5129956) – Microsoft Support](https://support.microsoft.com/help/5129956): Artículo KB para Exchange 2019 CU15 con indicación de ESU.

5.  [Description of version 2 of the security update for Microsoft Exchange Server 2019 CU14 (KB5129957) – Microsoft Support](https://support.microsoft.com/help/5129957): Artículo KB para Exchange 2019 CU14.

6.  [Description of version 2 of the security update for Microsoft Exchange Server 2016 CU23 October 2, 2026 (KB5129958) – Microsoft Support](https://support.microsoft.com/help/5129958): Artículo KB para Exchange 2016 CU23.

7.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Números de compilación de las SU de septiembre, de v2 del 2 de octubre de 2026 y de las SU de octubre del 13 de octubre de 2026.

8.  [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Versiones afectadas, corrección con las SU de octubre y reversión de la solución alternativa de reescritura de URL.

9.  [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Compilaciones SE afectadas, incluida v2, corrección con KB5128895 y sin más pasos tras la solución alternativa manual.

10. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Síntomas, corrección en el servicio y comandos para SettingOverride.

11. [V2 Security Updates Exchange 2016-SE (Sep2026) – EighTwOne](https://eightwone.com/2026/10/03/v2-security-updates-exchange-2016-se-sep2026/): Resumen de las compilaciones y KB de v2, indicación sobre paquetes específicos de CU.

12. [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-october-2026-exchange-server-security-updates/ba-p/4563014): SU de octubre como última actualización para Exchange 2016 y 2019, problemas conocidos corregidos (consultado mediante el feed RSS del Exchange Team Blog).

13. [Description of the security update for Microsoft Exchange Server Subscription Edition RTM October 13, 2026 (KB5128895) – Microsoft Support](https://support.microsoft.com/help/5128895): Lista de CVE y problemas corregidos de la SU de octubre para Exchange SE.
