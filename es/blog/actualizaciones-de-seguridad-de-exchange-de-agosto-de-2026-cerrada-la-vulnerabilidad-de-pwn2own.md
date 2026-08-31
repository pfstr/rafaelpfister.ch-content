---
title: "Actualizaciones de seguridad de Exchange de agosto de 2026: corregida la vulnerabilidad de Pwn2Own y desactivado OWA Light"
navTitle: "Exchange SU 08/2026"
description: "La SU de agosto corrige siete vulnerabilidades, incluido el exploit de Exchange demostrado en Pwn2Own 2026, y desactiva OWA Light de forma definitiva. Además, Microsoft explica por qué las SU de Exchange ahora se publican mensualmente y por qué Exchange SE CU1 sigue demorándose."
date: "2026-08-19"
kategorie: "Exchange OnPrem / híbrido"
timeToRead: "6 min de lectura"
themen:
  - exchange-updates
produkte:
  - "exchange"
protokolle:
  - "releases"
  - "powershell"
slug: "actualizaciones-de-seguridad-de-exchange-de-agosto-de-2026-cerrada-la-vulnerabilidad-de-pwn2own"
translationId: "article-b07bfd4074212673"
draft: false
translationOf: exchange-security-updates-august-2026
translationSourceHash: 41e10101798a88902017688d719457fce48959ba3acd2b3f1c757867b1b368d7
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T09:59:25.896Z
translationReview: required
url: https://rafaelpfister.ch/es/blog/actualizaciones-de-seguridad-de-exchange-de-agosto-de-2026-cerrada-la-vulnerabilidad-de-pwn2own
---

# Actualizaciones de seguridad de Exchange de agosto de 2026: corregida la vulnerabilidad de Pwn2Own y desactivado OWA Light

Microsoft publicó actualizaciones de seguridad (SU) para Exchange Server el 11 de agosto de 2026, por cuarto mes consecutivo. Las actualizaciones corrigen siete vulnerabilidades. Ninguna era conocida públicamente de antemano, ninguna se está explotando activamente según el estado actual y Microsoft clasifica la explotación de las siete como «Exploitation Less Likely». Aun así, no se trata de un Patch Tuesday rutinario, por tres motivos: la actualización corrige la vulnerabilidad de Exchange demostrada en el concurso de hacking Pwn2Own, **desactiva OWA Light definitivamente tras casi veinte años**, y el equipo de Exchange explicó posteriormente por qué el ritmo mensual seguirá siendo la norma por ahora.

## Para qué versiones de Exchange está disponible la actualización

Las SU están disponibles para las siguientes versiones:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121573, compilación 15.2.2562.46; como actualización pública disponible de forma regular.
- **Exchange Server 2019 CU15**: KB5121574, compilación 15.2.1748.49; solo a través del **programa ESU del período 2**.
- **Exchange Server 2019 CU14**: KB5121575, compilación 15.2.1544.44; solo a través de ESU del período 2.
- **Exchange Server 2016 CU23**: KB5121576, compilación 15.1.2507.72; solo a través de ESU del período 2.

La situación es la misma que en julio: Exchange 2016 y 2019 están fuera de soporte. Las SU de mayo a octubre de 2026 solo las reciben quienes estén inscritos en el programa ESU del período 2. Todos los demás permanecen sin parches, con catorce vulnerabilidades abiertas, algunas de alta gravedad; la migración a Exchange SE ya no admite más demora. Exchange Online ya está protegido; en entornos híbridos, la SU debe instalarse igualmente en todos los servidores Exchange, incluidos los servidores dedicados exclusivamente a administración y las máquinas en las que solo estén instaladas las Exchange Management Tools.

El problema conocido de los *mensajes wrapper* en buzones compartidos de entornos híbridos persiste también con la SU de agosto; según Microsoft, la corrección está prevista para una actualización futura. Al menos hay una aclaración tranquilizadora en los comentarios del anuncio de lanzamiento: quienes hayan establecido el SettingOverride documentado como solución alternativa **no** tienen que volver a crearlo tras instalar la SU de agosto. La actualización no modifica el override, tal como confirma allí el equipo de Exchange.

## Las siete vulnerabilidades de un vistazo

| CVE | Tipo | CVSS |
| --- | --- | --- |
| CVE-2026-62913 | Ejecución remota de código | 8.8 |
| CVE-2026-62911 | Elevación de privilegios | 8.0 |
| CVE-2026-62914 | Suplantación | 7.3 |
| CVE-2026-62910 | Elevación de privilegios | 7.2 |
| CVE-2026-62912 | Denegación de servicio | 6.5 |
| CVE-2026-62915 | Omitir una función de seguridad | 6.5 |
| CVE-2026-65813 | Elevación de privilegios | 6.5 |

Tres de ellas merecen una mirada más detallada.

**[CVE-2026-62913](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62913)** tiene, con CVSS 8.8, la puntuación más alta del mes: una ejecución remota de código que un atacante autenticado con permisos básicos puede desencadenar sin interacción alguna del usuario. Basta cualquier cuenta de buzón comprometida como punto de partida; en tiempos de phishing y credential stuffing, «autenticado» no es un obstáculo elevado.

**[CVE-2026-62911](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62911)** es la única vulnerabilidad del mes que Microsoft clasifica como *Critical* (elevación de privilegios, CVSS 8.0). Detrás hay más historia de la que revela el sobrio número: ante la pregunta de si el exploit de Exchange demostrado por Orange Tsai en **Pwn2Own 2026** ya había sido corregido, el equipo de Exchange remite precisamente a esta CVE en los comentarios del anuncio de lanzamiento. Con ello queda corregido el hallazgo del concurso: un motivo más para no dejar pendiente la SU de agosto, ya que las técnicas de Pwn2Own suelen publicarse con detalle tras vencer los períodos de embargo.

**[CVE-2026-62914](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62914)** (suplantación, CVSS 7.3) es el motivo directo para la desactivación de OWA Light, como veremos enseguida.

Las demás vulnerabilidades: CVE-2026-62910 (EoP, 7.2) ya requiere privilegios elevados, mientras que CVE-2026-62912 (DoS), CVE-2026-62915 (Security Feature Bypass) y CVE-2026-65813 (EoP) tienen CVSS 6.5. Los detalles se encuentran, como de costumbre, en la Security Update Guide (filtro «Server Software» para Exchange SE o «ESU» para 2016/2019).

## OWA Light: se acabó tras casi veinte años

### Qué cambia la actualización

Con la instalación de la SU de agosto, **OWA Light se desactiva permanentemente**: en cada servidor en el que se instale la actualización, o una posterior. Quienes accedan a la interfaz Light llegarán en adelante al Outlook on the web normal. La desactivación forma parte de la propia actualización y no se puede revertir mediante ningún interruptor; Microsoft la había anunciado unas semanas antes en una publicación de blog independiente.

OWA Light procede de la era de Exchange 2007: una interfaz web deliberadamente sencilla como alternativa para navegadores antiguos y conexiones lentas, oficialmente deprecated desde agosto de 2024. La razón de su retirada está motivada por la seguridad: una ruta de renderizado heredada separada junto al OWA moderno aumenta la complejidad y, con ella, la superficie de ataque; CVE-2026-62914 aporta la prueba concreta. Quienes hayan leído el [artículo de julio](/blog/exchange-security-updates-juli-2026) también lo recordarán: la mitigación de mayo para CVE-2026-42897 ya había dejado OWA Light inoperativo de forma colateral. Por tanto, la interfaz ya estaba en vías de desaparición.

### Quienes no puedan aplicar parches: desactivar OWA Light manualmente

Importante para quienes (todavía) no puedan instalar la SU de agosto, por ejemplo porque les falte la habilitación de ESU: Microsoft recomienda expresamente **desactivar manualmente** OWA Light en ese caso para mitigar CVE-2026-62914. Esto se hace mediante la política de buzón de OWA y la página de inicio de sesión:

```powershell
Get-OwaMailboxPolicy | Set-OwaMailboxPolicy -OWALightEnabled $false
Get-OwaVirtualDirectory | Set-OwaVirtualDirectory -LogonPageLightSelectionEnabled $false
```

El primer comando desactiva la versión Light para todos los buzones de la política correspondiente; el segundo elimina la opción «Usar la versión Light» de la página de inicio de sesión de OWA. Los cambios en el directorio virtual de OWA solo se aplican de forma fiable tras reciclar el grupo de aplicaciones de OWA o realizar un `iisreset`.

### Qué deberían comprobar ahora los administradores

La desactivación es trivial desde el punto de vista técnico, pero no siempre desde el organizativo: OWA Light era la solución de respaldo discreta para escenarios de nicho. Ahora conviene revisar los marcadores y las guías de helpdesk que tengan `?layout=light` codificado de forma fija, los dispositivos de quiosco y terminal con navegadores antiguos, así como las instrucciones internas para usuarios que utilizaban la versión Light por motivos de accesibilidad. El Outlook on the web moderno funciona en todos los navegadores actuales e incluye sus propias funciones de accesibilidad; pero quien no informe previamente a los usuarios afectados generará tickets.

## Por qué ahora se publica una SU cada mes y dónde está Exchange SE CU1

Dos días después del lanzamiento, el equipo de Exchange respondió en una publicación de blog notablemente abierta («Where is Exchange SE CU1 anyway?») a la pregunta que se hacen muchos administradores. La versión corta: Microsoft utiliza herramientas de IA en toda la empresa para encontrar vulnerabilidades en sus propios productos. Los equipos, incluido Exchange, están trabajando actualmente los hallazgos notificados: validarlos, reproducirlos, corregirlos, probar regresiones y entregarlos mensualmente. Desde mayo de 2026 se ha publicado así una SU de Exchange cada mes, y Microsoft afirma expresamente que este ritmo acelerado continuará.

El esperado **CU1 para Exchange SE** se retrasa precisamente por este motivo. Anunciado originalmente para el primer semestre de 2026 y pospuesto después al segundo, ya ni siquiera tiene una fecha objetivo. Microsoft solo quiere publicar CU1 cuando haya un mes sin una entrega urgente de seguridad entre medias; un CU que fuese superado de inmediato por una SU supondría trabajo de actualización duplicado para muchas organizaciones. Hasta entonces, la carga mensual de seguridad se incorpora continuamente a la compilación interna de CU1.

Para la práctica, esto significa dos cosas. En primer lugar: esperar a CU1 no es una estrategia, ni para la migración a SE ni para instalar las SU. En segundo lugar: una **ventana de mantenimiento mensual** para Exchange debe pasar a formar parte fija del calendario operativo, como ocurre desde hace tiempo con los servidores Windows.

## Instalación y tareas posteriores

El procedimiento sigue siendo el habitual: primero, inventariar con el [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) qué servidores se encuentran en qué nivel de CU/SU y si quedan pasos manuales pendientes. Después, instalar la SU (si el CU está desactualizado, el [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) muestra la ruta), reiniciar el servidor y comprobar que todos los servicios de Exchange se hayan iniciado correctamente. Si hay servicios *desactivados*, la instalación se interrumpió; en ese caso ayuda la solución alternativa documentada en el artículo de soporte de Microsoft sobre el error de versión de archivo o el [script SetupAssist](https://aka.ms/ExSetupAssist). Para finalizar, ejecutar de nuevo el Health Checker.

Las SU son acumulativas: quien se haya saltado la SU de julio puede instalar directamente la SU de agosto. Y para entornos híbridos se aplica el complemento conocido: si se cambia el certificado de autenticación tras la instalación de la SU, se debe ejecutar de nuevo el Hybrid Configuration Wizard.

Sigue vigente una tarea posterior de julio: quien aún tenga activa la mitigación para CVE-2026-42897 (M2.1.0) debería eliminarla ahora; el procedimiento correcto se explica en el [artículo sobre la SU de julio](/blog/exchange-security-updates-juli-2026).

## Procedimiento recomendado

En resumen: instalar sin demora la SU de agosto en todos los servidores Exchange y máquinas con Management Tools; la vulnerabilidad de Pwn2Own y la RCE con 8.8 son razones suficientes para no esperar al próximo Patch Tuesday. Quienes no puedan aplicar el parche de inmediato pueden desactivar OWA Light manualmente como medida inmediata frente a CVE-2026-62914. Antes de desactivar OWA Light, identificar e informar a los grupos de usuarios afectados (marcadores antiguos, navegadores de quiosco, flujos de trabajo de accesibilidad). Después, ejecutar el Health Checker, completar las tareas pendientes de julio y planificar una ventana mensual de mantenimiento de Exchange, porque el ritmo se mantiene.

## Fuentes

1.  [Released: August 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-august-2026-exchange-server-security-updates/4543951): Anuncio oficial de lanzamiento con las versiones compatibles, la nota sobre OWA Light, problemas conocidos y preguntas frecuentes; en los comentarios, las confirmaciones sobre la corrección de Pwn2Own (CVE-2026-62911) y el SettingOverride de wrapper que continúa vigente.

2.  [Upcoming retirement of OWA Light in Exchange Server – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/upcoming-retirement-of-owa-light-in-exchange-server/4534943): El anuncio previo de la desactivación, incluida la recomendación de Microsoft de desactivar manualmente OWA Light si no se aplica el parche.

3.  [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837): El equipo de Exchange sobre la detección de vulnerabilidades asistida por IA, el ritmo mensual sostenido de las SU y el retraso de CU1.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Referencia para los números de compilación de las SU de agosto.

5.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Condiciones y duración (de mayo a octubre de 2026) del programa ESU.

6.  [Wrapper messages appear in shared mailbox inbox in hybrid environments – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): El problema híbrido conocido desde junio, incluida la solución alternativa SettingOverride.

7.  [Neue Sicherheitsupdates für Exchange Server (August 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-august-2026/): Desglose en alemán de las siete CVE con valores CVSS y compilaciones.

8.  [Set-OwaMailboxPolicy (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-owamailboxpolicy): El parámetro `OWALightEnabled` para desactivar manualmente la versión Light.

9.  [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventario de niveles de CU/SU y pasos manuales pendientes antes y después de la instalación.
