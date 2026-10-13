---
title: "Actualización de seguridad de Exchange de septiembre de 2026 v2: corrección anticipada para CVE-2026-96940"
navTitle: "Exchange SU 09/2026 v2"
description: "El 2 de octubre de 2026, Microsoft publicó la versión 2 de las SU de septiembre para Exchange SE, 2019 y 2016. También corrige CVE-2026-96940, una vulnerabilidad de elevación de privilegios con CVSS 8.8 y la valoración «Exploitation More Likely». Microsoft recomienda instalar v2 lo antes posible, también en servidores con la primera SU de septiembre."
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
translationSourceHash: 7b6f57477ce51bb986d8fe10711a2f715adc0bebe060c97c99fd6d9cc7c2da15
translationModel: gpt-5.6-terra
translatedAt: 2026-10-10T10:03:57.057Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/actualizacion-de-seguridad-de-exchange-de-septiembre-de-2026-v2-correccion-anticipada-para-cve
---

# Actualización de seguridad de Exchange de septiembre de 2026 v2: corrección anticipada para CVE-2026-96940

El 2 de octubre de 2026, Microsoft publicó una **versión 2 (v2)** de las actualizaciones de seguridad (SU) de septiembre para Exchange Server SE, Exchange 2019 y Exchange 2016. Según el anuncio, la única diferencia con la primera versión del 8 de septiembre es que v2 también corrige **CVE-2026-96940**. La vulnerabilidad no es pública ni está siendo explotada; Microsoft la detectó internamente. Hay dos aspectos inusuales: según Microsoft, esta actualización se publicó antes de lo previsto, y la Security Update Guide clasifica la explotación como **«Exploitation More Likely»**. Las nueve vulnerabilidades de la primera versión, el problema de Wrapper corregido y los problemas conocidos se describen en el [artículo sobre la SU de septiembre](/blog/exchange-security-updates-september-2026); este artículo trata las novedades de v2.

## Para qué versiones de Exchange está disponible la actualización

Las SU v2 del 2 de octubre de 2026 están disponibles para las siguientes versiones:

- **Exchange Server Subscription Edition (SE) RTM**: KB5129955, compilación 15.2.2562.53; disponible públicamente.
- **Exchange Server 2019 CU15**: KB5129956, compilación 15.2.1748.53; solo a través del **programa ESU Period 2**.
- **Exchange Server 2019 CU14**: KB5129957, compilación 15.2.1544.48; solo a través de ESU Period 2.
- **Exchange Server 2016 CU23**: KB5129958, compilación 15.1.2507.75; solo a través de ESU Period 2.

Las actualizaciones son específicas de cada CU: el paquete para CU15 no puede instalarse en CU14. Exchange 2016 y 2019 ya no reciben soporte. Según las FAQ del anuncio, solo las organizaciones del programa ESU Period 2, vigente de mayo a octubre de 2026, reciben actualizaciones publicadas después de mayo de 2026 para estas versiones; Microsoft recomienda a todas las demás migrar a Exchange SE lo antes posible. Para los entornos híbridos con servidores obsoletos, se añade la [aplicación de transporte de Exchange Online](/blog/exchange-online-transport-enforcement-hybrid-server).

Puede comprobar si sus servidores ya están en el nivel v2 mediante la lista de [números de compilación de Exchange](/tools/exchange-builds).

## La vulnerabilidad de un vistazo

| CVE | Tipo | CVSS |
| --- | --- | --- |
| CVE-2026-96940 | Elevación de privilegios | 8.8 |

Microsoft describe **[CVE-2026-96940](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940)** como una autorización insuficiente en Exchange Server, mediante la cual un atacante autenticado puede ampliar sus privilegios a través de la red. El vector CVSS indica requisitos bajos: ataque a través de la red, baja complejidad, una cuenta con pocos privilegios y ninguna interacción del usuario. Según las FAQ de la Security Update Guide, un atacante que tenga éxito obtiene acceso no autorizado a los buzones de otros usuarios de la misma organización y puede leer sus correos, incluidos los adjuntos; el acceso queda limitado a la propia organización. Microsoft clasifica la vulnerabilidad como *Important* y la puntuación temporal es 7.7.

La valoración «Exploitation More Likely» es la diferencia más importante respecto a las nueve CVE de la primera versión, que Microsoft calificó todas como «Exploitation Less Likely». El punto de partida de un ataque es cualquier cuenta de usuario válida, por ejemplo, tras un phishing exitoso. Microsoft corrigió Exchange Online en el lado del servidor; allí no hay que hacer nada. Microsoft no menciona ninguna mitigación ni solución alternativa para servidores sin v2 (a fecha del 7 de octubre de 2026).

Las demás CVE de los artículos KB de v2 son las mismas que en la primera versión: CVE-2026-55007, CVE-2026-69355, CVE-2026-69356, CVE-2026-69361, CVE-2026-69375, CVE-2026-69378, CVE-2026-69382 y CVE-2026-69641. Al igual que en las KB de septiembre, falta CVE-2026-69380, que según la Security Update Guide también se corrige con la SU de septiembre.

## Por qué existe una v2 y qué debe hacer

En las FAQ del anuncio, Microsoft responde así a la pregunta sobre la secuencia inusual: la actualización con CVE-2026-96940 se publicó antes de la fecha prevista, y Microsoft recomienda revisar las indicaciones de implementación e instalar la actualización en la primera oportunidad posible. Microsoft no explica por qué se adelantó la fecha.

De ello se desprende lo siguiente para la práctica:

1. **Los servidores con la SU del 8 de septiembre** (compilaciones 15.2.2562.49, 15.2.1748.51, 15.2.1544.46, 15.1.2507.73) no están protegidos frente a CVE-2026-96940. La Security Update Guide enumera exclusivamente las cuatro compilaciones v2 como corrección para esta CVE. Estos servidores necesitan v2.

2. **Los servidores con una versión anterior** pueden instalar v2 directamente. Según las FAQ, las SU son acumulativas; quienes estén en una CU compatible solo instalan la SU más reciente.

3. **Las máquinas con las Exchange Management Tools** y los servidores dedicados exclusivamente a la administración también reciben v2; Microsoft lo recomienda para todas las SU, incluso en entornos híbridos.

Según KB5129955, el archivo de instalación para Exchange SE se llama `ExchangeSubscriptionEdition-KB5129955-x64-en.exe`; está disponible a través de Microsoft Update Catalog y Download Center, donde figura con la denominación «SU10V2».

## Problemas conocidos

Los problemas conocidos de la primera versión también se aplican a v2. A fecha del 7 de octubre de 2026:

**Los calendarios publicados (.ics) devuelven HTTP 500 a las aplicaciones de calendario** (todas las versiones). El problema existe desde la SU de agosto y sigue figurando como problema conocido en las cuatro KB v2. Microsoft todavía lo está investigando y describe como solución temporal una regla de reescritura de URL en el sitio «Exchange Back End»; los pasos se encuentran en el [artículo de soporte KB5126672](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672) y en el [artículo sobre la SU de septiembre](/blog/exchange-security-updates-september-2026). La regla debe configurarse en cada servidor y, según Microsoft, volver a comprobarse después de futuras actualizaciones.

**Bloqueo mutuo de ContentEngine debido a archivos WordBreaker coreanos ausentes** (solo Exchange SE). Según [KB5130098](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098), ambas versiones están expresamente afectadas: la compilación 15.2.2562.49 y la compilación v2 15.2.2562.53. Por tanto, v2 no corrige el problema. Los síntomas incluyen resultados de búsqueda ausentes, entrega de correo retrasada y clientes Outlook o MAPI bloqueados o desconectados. Según la KB, se ven afectados los entornos que procesan correos con contenido en coreano; no depende del idioma de los propios usuarios, pues el texto coreano también llega a organizaciones sin correspondencia en coreano, por ejemplo, como spam. Health Checker no detecta los archivos ausentes. Encontrará el script de comprobación, los requisitos y la solución manual paso a paso en el [artículo sobre la SU de septiembre](/blog/exchange-security-updates-september-2026#workaround-für-den-wordbreaker-deadlock); Microsoft sigue investigando el problema.

**Libre/ocupado para buzones delegados en entornos híbridos con configuración exclusiva de Graph API** (solo Exchange SE). Aquí las fuentes se contradicen: el anuncio de v2 incluye el problema entre los problemas corregidos, KB5129955 sigue enumerándolo como problema conocido y el [artículo de soporte KB5127092](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092) indica el estado «Microsoft is investigating this issue» sin mencionar una corrección. Si ha establecido el SettingOverride `EnableRouteThroughMSGraphFeature` descrito allí como solución alternativa, retírelo tras instalar v2 solo después de realizar pruebas, mientras Microsoft no publique instrucciones al respecto.

## Instalación y tareas posteriores

En el anuncio, Microsoft indica el procedimiento habitual: realizar un inventario con [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker), determinar la ruta con [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) si la CU está desactualizada, reiniciar el servidor después de la instalación y comprobar que todos los servicios de Exchange se han iniciado. En caso de errores durante o después de la instalación, Microsoft remite al [script SetupAssist](https://aka.ms/ExSetupAssist). Para las compilaciones v2 de CVE-2026-96940, la Security Update Guide no indica que sea necesario reiniciar; no obstante, el anuncio recomienda reiniciar después de la instalación.

Para los entornos híbridos, las FAQ indican lo siguiente: si se cambia el certificado de autenticación después de instalar una SU, debe ejecutarse de nuevo Hybrid Configuration Wizard. En Windows Server 2025, las SU de Exchange instaladas no aparecen en el Panel de control; Microsoft desaconseja en general desinstalar las SU y remite a un artículo de soporte específico para este caso.

De los meses anteriores siguen pendientes estas tareas posteriores, si aún no se han realizado: eliminar el Wrapper-SettingOverride `DisableBlockSharedAndUserMailboxHeaders` (véase el [artículo de septiembre](/blog/exchange-security-updates-september-2026)) y comprobar si la mitigación de CVE-2026-42897 (M2.1.0) sigue activa (véase el [artículo sobre la SU de julio](/blog/exchange-security-updates-juli-2026)).

## Procedimiento recomendado

Instale v2 sin demora en todos los servidores Exchange y en las máquinas con Management Tools, incluso donde ya se esté ejecutando la SU de septiembre: solo las compilaciones v2 corrigen CVE-2026-96940, una sola cuenta de usuario comprometida basta para acceder a buzones ajenos y Microsoft considera la explotación más probable que la de todas las demás CVE de septiembre. Después, compruebe con Health Checker el nivel de compilación, verifique en Exchange SE con el script de comprobación del artículo de septiembre si faltan los archivos WordBreaker y mantenga las soluciones alternativas existentes hasta que Microsoft documente su retirada. El programa ESU para Exchange 2016 y 2019 termina en octubre de 2026; después ya no se publicarán actualizaciones para estas versiones.

## Fuentes

1.  [Released: September 2026 V2 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/t5/exchange-team-blog/released-september-2026-v2-exchange-server-security-updates/ba-p/4561718): Anuncio de v2 del 2 de octubre de 2026, con las diferencias respecto a la primera versión, las FAQ sobre la publicación anticipada, problemas conocidos, problemas corregidos y el procedimiento de instalación (consultado a través del canal RSS del Exchange Team Blog).

2.  [CVE-2026-96940 – Security Update Guide, Microsoft Security Response Center](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-96940): Tipo, CVSS 8.8/7.7 con vector, gravedad, valoración de explotación, FAQ y las cuatro compilaciones v2 como corrección.

3.  [Description of version 2 of the security update for Microsoft Exchange Server Subscription Edition RTM October 2, 2026 (KB5129955) – Microsoft Support](https://support.microsoft.com/help/5129955): Lista de CVE, problemas conocidos, problemas corregidos y archivo de instalación para Exchange SE.

4.  [Description of version 2 of the security update for Microsoft Exchange Server 2019 CU15 (KB5129956) – Microsoft Support](https://support.microsoft.com/help/5129956): Artículo KB para Exchange 2019 CU15 con indicación sobre ESU.

5.  [Description of version 2 of the security update for Microsoft Exchange Server 2019 CU14 (KB5129957) – Microsoft Support](https://support.microsoft.com/help/5129957): Artículo KB para Exchange 2019 CU14.

6.  [Description of version 2 of the security update for Microsoft Exchange Server 2016 CU23 October 2, 2026 (KB5129958) – Microsoft Support](https://support.microsoft.com/help/5129958): Artículo KB para Exchange 2016 CU23.

7.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Números de compilación de las SU de septiembre y de v2 del 2 de octubre de 2026.

8.  [Published calendar (.ics) returns HTTP 500 for calendar applications – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5126672): Versiones afectadas, estado y solución alternativa de reescritura de URL.

9.  [ContentEngine deadlock because of missing Korean WordBreaker rule files – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5130098): Compilaciones SE afectadas, incluida v2, criterios de comprobación y solución manual.

10. [Availability (free/busy) fails for delegated mailboxes in Exchange hybrid deployments using Graph API only – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/update/2026/5127092): Síntomas, solución alternativa mediante SettingOverride y estado de la investigación.

11. [V2 Security Updates Exchange 2016-SE (Sep2026) – EighTwOne](https://eightwone.com/2026/10/03/v2-security-updates-exchange-2016-se-sep2026/): Resumen de las compilaciones v2 y las KB, con indicación sobre los paquetes específicos de cada CU.
