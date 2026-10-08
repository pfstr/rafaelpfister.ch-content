---
title: "Actualización de seguridad de Exchange de septiembre de 2026 v2: corrección anticipada para CVE-2026-96940"
navTitle: "Exchange SU 09/2026 v2"
description: "El 2 de octubre de 2026, Microsoft publicó una versión 2 de las SU de septiembre para Exchange SE, 2019 y 2016. También corrige CVE-2026-96940, una vulnerabilidad de elevación de privilegios con CVSS 8.8 y la evaluación «Exploitation More Likely». Microsoft recomienda instalar v2 lo antes posible, incluso en servidores con la primera SU de septiembre."
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
url: https://rafaelpfister.ch/es/blog/actualizacion-de-seguridad-de-exchange-de-septiembre-de-2026-v2-correccion-anticipada-para-cve
translationSourceHash: bebc65ad45b679575e83f58eca5401945194e0aa5040a55d38e1b11f0cd72fa4
translationModel: gpt-5.6-terra
translatedAt: 2026-10-08T10:48:39.168Z
translationReview: automatic
---

# Actualización de seguridad de Exchange de septiembre de 2026 v2: corrección anticipada para CVE-2026-96940

El 2 de octubre de 2026, Microsoft publicó una **versión 2 (v2)** de las actualizaciones de seguridad (SU) de septiembre para Exchange Server SE, Exchange 2019 y Exchange 2016. Según el anuncio, la única diferencia respecto a la primera versión del 8 de septiembre es que v2 también corrige **CVE-2026-96940**. La vulnerabilidad no es conocida públicamente ni está siendo explotada; Microsoft la descubrió internamente. Hay dos aspectos inusuales: según Microsoft, publicó esta actualización antes de lo previsto, y la Security Update Guide califica la explotación como **«Exploitation More Likely»**. Las nueve vulnerabilidades de la primera versión, el problema del wrapper corregido y los problemas conocidos se describen en el [artículo sobre la SU de septiembre](/blog/exchange-security-updates-september-2026); este artículo trata las novedades de v2.

## Para qué versiones de Exchange está disponible la actualización

Las SU v2 del 2 de octubre de 2026 están disponibles para las siguientes versiones:

- **Exchange Server Subscription Edition (SE) RTM**: KB5129955, compilación 15.2.2562.53; disponible públicamente.
- **Exchange Server 2019 CU15**: KB5129956, compilación 15.2.1748.53; solo mediante el **programa Period 2 ESU**.
- **Exchange Server 2019 CU14**: KB5129957, compilación 15.2.1544.48; solo mediante Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5129958, compilación 15.1.2507.75; solo mediante Period 2 ESU.

Las actualizaciones son específicas de cada CU: el paquete para CU15 no se puede instalar en CU14. Exchange 2016 y 2019 están fuera de soporte. Según las preguntas frecuentes del anuncio, solo las organizaciones incluidas en el programa Period 2 ESU, vigente de mayo a octubre de 2026, reciben actualizaciones publicadas después de mayo de 2026 para estas versiones; Microsoft recomienda a todas las demás migrar a Exchange SE lo antes posible. Para los entornos híbridos con servidores obsoletos, se añade la [aplicación de transporte de Exchange Online](/blog/exchange-online-transport-enforcement-hybrid-server).

Puede comprobar si sus servidores ya están en el nivel v2 mediante la lista de [números de compilación de Exchange](/tools/exchange-builds).

## La vulnerabilidad de un vistazo

| CVE | Tipo | CVSS |
| --- | --- | --- |
| CVE-2026-96940 | Elevación de privilegios | 8.8 |

Microsoft describe **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940)** como una autorización débil en Exchange Server que permite a un atacante autenticado ampliar sus privilegios a través de la red. El vector CVSS indica requisitos bajos: ataque por red, baja complejidad, una cuenta con pocos privilegios y sin interacción del usuario. Según las preguntas frecuentes de la Security Update Guide, un atacante que tenga éxito obtiene acceso no autorizado a los buzones de otros usuarios de la misma organización y puede leer sus correos electrónicos y archivos adjuntos; el acceso queda limitado a su propia organización. Microsoft clasifica la vulnerabilidad como *Important* y la puntuación temporal es 7.7.

La valoración «Exploitation More Likely» es la diferencia más importante respecto a los nueve CVE de la primera versión, que Microsoft calificó todos como «Exploitation Less Likely». El punto de partida para un ataque es cualquier cuenta de usuario válida, por ejemplo, tras un phishing exitoso. Microsoft corrigió Exchange Online del lado del servidor; no hay que hacer nada allí. Microsoft no indica ninguna mitigación ni solución alternativa para servidores sin v2 (a fecha de 7 de octubre de 2026).

Los demás CVE de los artículos KB de v2 son los mismos que en la primera versión: CVE-2026-55007, CVE-2026-69355, CVE-2026-69356, CVE-2026-69361, CVE-2026-69375, CVE-2026-69378, CVE-2026-69382 y CVE-2026-69641. Al igual que en los KB de septiembre, falta CVE-2026-69380, que según la Security Update Guide también se corrige con la SU de septiembre.

## Por qué existe una v2 y qué debe hacer

En las preguntas frecuentes del anuncio, Microsoft responde así a la cuestión de la secuencia inusual: la actualización con CVE-2026-96940 se publicó antes de la fecha prevista, y Microsoft recomienda revisar las indicaciones de implementación e instalar la actualización lo antes posible. Microsoft no explica por qué se adelantó la fecha.

En la práctica, esto significa:

1. **Los servidores con la SU del 8 de septiembre** (compilaciones 15.2.2562.49, 15.2.1748.51, 15.2.1544.46, 15.1.2507.73) no están protegidos frente a CVE-2026-96940. La Security Update Guide solo enumera las cuatro compilaciones v2 como corrección para este CVE. Estos servidores necesitan v2.

2. **Los servidores con una versión anterior** pueden instalar v2 directamente. Según las preguntas frecuentes, las SU son acumulativas; quien esté en una CU compatible solo instala la SU más reciente.

3. **Las máquinas con Exchange Management Tools** y los servidores de administración dedicados también reciben v2; Microsoft lo recomienda para todas las SU, también en entornos híbridos.

Según KB5129955, el archivo de instalación para Exchange SE se llama `ExchangeSubscriptionEdition-KB5129955-x64-en.exe`; está disponible a través de Microsoft Update Catalog y Download Center, donde figura como «SU10V2».

## Problemas conocidos

Los problemas conocidos de la primera versión también se aplican a v2. Situación a 7 de octubre de 2026:

**Los calendarios publicados (.ics) devuelven HTTP 500 en aplicaciones de calendario** (todas las versiones). El problema existe desde la SU de agosto y sigue figurando como problema conocido en los cuatro KB de v2. Microsoft aún lo está investigando y describe como solución alternativa temporal una regla de reescritura de URL en el sitio «Exchange Back End»; los pasos se encuentran en el [artículo de soporte KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672) y en el [artículo sobre la SU de septiembre](/blog/exchange-security-updates-september-2026). La regla debe configurarse en cada servidor y, según Microsoft, debe revisarse nuevamente después de futuras actualizaciones.

**Interbloqueo de ContentEngine debido a archivos WordBreaker coreanos faltantes** (solo Exchange SE). Según [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098), ambas versiones están expresamente afectadas: la compilación 15.2.2562.49 y la compilación v2 15.2.2562.53. Por tanto, v2 no corrige el problema. Los síntomas son resultados de búsqueda ausentes, entrega de correo retrasada y clientes Outlook o MAPI bloqueados o desconectados; según el anuncio, afecta a organizaciones con correos en coreano. La solución alternativa manual (dos archivos de reglas de SQL Server 2025 Express RTM, verificación mediante hash SHA256) se encuentra en el artículo de soporte; Microsoft sigue investigando el problema.

**Libre/ocupado para buzones delegados en entornos híbridos con configuración exclusiva de Graph API** (solo Exchange SE). Aquí las fuentes se contradicen: el anuncio de v2 incluye el problema entre los problemas corregidos, KB5129955 sigue enumerándolo como problema conocido, y el [artículo de soporte KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) indica el estado «Microsoft is investigating this issue» sin referencia a una corrección. Si ha configurado el SettingOverride `EnableRouteThroughMSGraphFeature` allí descrito como solución alternativa, elimínelo tras instalar v2 solo después de realizar una prueba, mientras Microsoft no publique instrucciones al respecto.

## Instalación y tareas posteriores

Microsoft menciona en el anuncio el procedimiento habitual: inventariar con el [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), determinar la ruta en caso de una CU obsoleta con el [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard), reiniciar el servidor tras la instalación y comprobar que todos los servicios de Exchange se hayan iniciado. En caso de errores durante o después de la instalación, Microsoft remite al [script SetupAssist](https://aka.ms/ExSetupAssist). La Security Update Guide no indica que sea necesario reiniciar para las compilaciones v2 en relación con CVE-2026-96940; no obstante, el anuncio recomienda reiniciar tras la instalación.

Para los entornos híbridos, según las preguntas frecuentes, se aplica lo siguiente: si se cambia el certificado de autenticación tras instalar una SU, hay que ejecutar de nuevo el Hybrid Configuration Wizard. En Windows Server 2025, las SU de Exchange instaladas no aparecen en el Panel de control; Microsoft desaconseja en general desinstalar las SU y remite a un artículo de soporte específico para este caso.

De los meses anteriores, quedan pendientes estas tareas si aún no se han realizado: eliminar el SettingOverride del wrapper `DisableBlockSharedAndUserMailboxHeaders` (véase el [artículo de septiembre](/blog/exchange-security-updates-september-2026)) y comprobar si la mitigación para CVE-2026-42897 (M2.1.0) sigue activa (véase el [artículo sobre la SU de julio](/blog/exchange-security-updates-juli-2026)).

## Procedimiento recomendado

Instale v2 cuanto antes en todos los servidores Exchange y en las máquinas con Management Tools, incluso donde ya esté instalada la SU de septiembre: solo las compilaciones v2 corrigen CVE-2026-96940, una única cuenta de usuario comprometida basta para acceder a buzones ajenos, y Microsoft considera más probable la explotación que para todos los demás CVE de septiembre. Después, compruebe el nivel de compilación con Health Checker, revise el problema de WordBreaker en Exchange SE y mantenga las soluciones alternativas existentes hasta que Microsoft documente su retirada. Para Exchange 2016 y 2019, el programa ESU finaliza en octubre de 2026; después ya no se publicarán actualizaciones para estas versiones.

## Fuentes

1.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Anuncio de v2 del 2 de octubre de 2026 con las diferencias respecto a la primera versión, las preguntas frecuentes sobre la publicación anticipada, problemas conocidos, problemas corregidos y procedimiento de instalación (consultado a través del feed RSS del Exchange Team Blog).

2.  [CVE-2026-96940 – Security Update Guide, Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940): Tipo, CVSS 8.8/7.7 con vector, nivel de gravedad, evaluación de explotación, preguntas frecuentes y las cuatro compilaciones v2 como corrección.

3.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): Lista de CVE, problemas conocidos, problemas corregidos y archivo de instalación para Exchange SE.

4.  [Description of version 2 of the security update for Microsoft Exchange Server 2019 CU15 (KB5129956) – Microsoft Support](https://support.microsoft.com/help/5129956): Artículo KB para Exchange 2019 CU15 con indicación sobre ESU.

5.  [Description of version 2 of the security update for Microsoft Exchange Server 2019 CU14 (KB5129957) – Microsoft Support](https://support.microsoft.com/help/5129957): Artículo KB para Exchange 2019 CU14.

6.  [Description of version 2 of the security update for Microsoft Exchange Server 2016 CU23 October 2, 2026 (KB5129958) – Microsoft Support](https://support.microsoft.com/help/5129958): Artículo KB para Exchange 2016 CU23.

7.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Números de compilación de las SU de septiembre y de v2 del 2 de octubre de 2026.

8.  [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Versiones afectadas, estado y solución alternativa de reescritura de URL.

9.  [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Compilaciones SE afectadas, incluida v2, y solución alternativa manual.

10. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Síntomas, solución alternativa con SettingOverride y estado de la investigación.

11. [V2 Security Updates Exchange 2016-SE (Sep2026) – EighTwOne](https://eightwone.com/2026/10/03/v2-security-updates-exchange-2016-se-sep2026/): Resumen de las compilaciones y KB de v2, indicación sobre paquetes específicos de CU.
