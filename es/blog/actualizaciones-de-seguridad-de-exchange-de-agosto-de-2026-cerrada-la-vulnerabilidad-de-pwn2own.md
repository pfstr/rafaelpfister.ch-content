---
title: "Actualizaciones de seguridad de Exchange de agosto de 2026: cerrada la vulnerabilidad de Pwn2Own, desactivado OWA Light"
navTitle: "Exchange SU 08/2026"
description: "La SU de agosto corrige siete vulnerabilidades, incluido el exploit de Exchange demostrado en Pwn2Own 2026, y desactiva OWA Light de forma definitiva. Microsoft también explica por qué las SU de Exchange ahora se publican mensualmente y por qué Exchange SE CU1 sigue retrasándose."
date: "2026-08-19"
kategorie: "Exchange OnPrem / Hybrid"
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
url: https://rafaelpfister.ch/es/blog/actualizaciones-de-seguridad-de-exchange-de-agosto-de-2026-cerrada-la-vulnerabilidad-de-pwn2own
translationSourceHash: a41c24b533c3b19bf6226ac5d16e7b9668d83d13b53588da7109f5567e79db51
translationModel: gpt-5.6-terra
translatedAt: 2026-08-20T04:04:35.567Z
translationReview: required
---

# Actualizaciones de seguridad de Exchange de agosto de 2026: cerrada la vulnerabilidad de Pwn2Own, desactivado OWA Light

Microsoft publicó el 11 de agosto de 2026 actualizaciones de seguridad (SU) para Exchange Server, por cuarto mes consecutivo. Las actualizaciones corrigen siete vulnerabilidades. Ninguna se conocía públicamente de antemano, ninguna está siendo explotada activamente según el estado actual y Microsoft clasifica la explotación de las siete como «Exploitation Less Likely». Sin embargo, no es un Patch Tuesday rutinario, por tres motivos: la actualización corrige la vulnerabilidad de Exchange demostrada en el concurso de hacking Pwn2Own, **desactiva OWA Light definitivamente después de casi veinte años**, y el equipo de Exchange ha explicado posteriormente por qué el ritmo mensual seguirá siendo la norma por ahora.

## Para qué versiones de Exchange está disponible la actualización

Las SU están disponibles para las siguientes versiones:

- **Exchange Server Subscription Edition (SE) RTM**: KB5121573, compilación 15.2.2562.46, como actualización pública disponible de forma regular.
- **Exchange Server 2019 CU15**: KB5121574, compilación 15.2.1748.49, solo mediante el **programa Period 2 ESU**.
- **Exchange Server 2019 CU14**: KB5121575, compilación 15.2.1544.44, solo mediante Period 2 ESU.
- **Exchange Server 2016 CU23**: KB5121576, compilación 15.1.2507.72, solo mediante Period 2 ESU.

La situación es la misma que en julio: Exchange 2016 y 2019 están fuera de soporte. Las SU de mayo a octubre de 2026 solo las reciben quienes estén inscritos en el programa Period 2 ESU. Todos los demás permanecen sin parches, con catorce vulnerabilidades abiertas, algunas de alta gravedad, por lo que la migración a Exchange SE ya no admite demora. Exchange Online ya está protegido; en entornos híbridos, la SU debe instalarse igualmente en todos los servidores Exchange, incluidos los servidores dedicados exclusivamente a la administración y las máquinas donde solo estén instaladas las Exchange Management Tools.

El problema conocido de los *mensajes wrapper* en buzones compartidos de entornos híbridos continúa incluso con la SU de agosto; según Microsoft, la corrección está prevista para una próxima actualización. Al menos hay una buena noticia en los comentarios del anuncio de lanzamiento: quienes hayan configurado el SettingOverride documentado como solución temporal **no** tienen que crearlo de nuevo después de instalar la SU de agosto; la actualización deja el override intacto, tal como confirma allí el equipo de Exchange.

## Las siete vulnerabilidades de un vistazo

| CVE | Tipo | CVSS |
| --- | --- | --- |
| CVE-2026-62913 | Remote Code Execution | 8.8 |
| CVE-2026-62911 | Elevation of Privilege | 8.0 |
| CVE-2026-62914 | Spoofing | 7.3 |
| CVE-2026-62910 | Elevation of Privilege | 7.2 |
| CVE-2026-62912 | Denial of Service | 6.5 |
| CVE-2026-62915 | Security Feature Bypass | 6.5 |
| CVE-2026-65813 | Elevation of Privilege | 6.5 |

Tres de ellas merecen una mirada más detallada.

**[CVE-2026-62913](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62913)** tiene, con CVSS 8.8, la puntuación más alta del mes: una ejecución remota de código que un atacante autenticado con permisos básicos puede desencadenar sin ninguna interacción del usuario. Basta cualquier cuenta de buzón comprometida como punto de partida; en tiempos de phishing y credential stuffing, «autenticado» no supone un gran obstáculo.

**[CVE-2026-62911](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62911)** es la única vulnerabilidad del mes que Microsoft clasifica como *Critical* (Elevation of Privilege, CVSS 8.0). Hay más historia detrás de ella de lo que revela el sobrio número: a la pregunta de si ya se había corregido el exploit de Exchange demostrado por Orange Tsai en **Pwn2Own 2026**, el equipo de Exchange remite precisamente a esta CVE en los comentarios del anuncio de lanzamiento. Con ello queda cerrada la vulnerabilidad del concurso, otro motivo para no dejar pendiente la SU de agosto, ya que las técnicas de Pwn2Own suelen publicarse en detalle tras expirar los plazos de embargo.

**[CVE-2026-62914](https://msrc.microsoft.com/update-guide/vulnerability/CVE-2026-62914)** (Spoofing, CVSS 7.3) es el motivo directo de la desactivación de OWA Light; más sobre ello a continuación.

Las demás vulnerabilidades: CVE-2026-62910 (EoP, 7.2) ya requiere privilegios elevados, mientras que CVE-2026-62912 (DoS), CVE-2026-62915 (Security Feature Bypass) y CVE-2026-65813 (EoP) tienen una puntuación CVSS de 6.5. Como de costumbre, los detalles se encuentran en la Security Update Guide (filtre por «Server Software» para Exchange SE o por «ESU» para 2016/2019).

## OWA Light: se acabó después de casi veinte años

### Qué cambia la actualización

Al instalar la SU de agosto, **OWA Light se desactiva permanentemente**, en todos los servidores donde se instale esta actualización o una posterior. Quienes accedan a la interfaz Light serán redirigidos en adelante al Outlook on the web normal. La desactivación forma parte de la propia actualización y no puede revertirse mediante ningún interruptor; Microsoft la había anunciado unas semanas antes en una publicación independiente de blog.

OWA Light procede de la época de Exchange 2007: una interfaz web deliberadamente sencilla como alternativa para navegadores antiguos y conexiones lentas, oficialmente deprecated desde agosto de 2024. El motivo de su retirada está impulsado por la seguridad: una ruta de renderizado heredada separada junto al OWA moderno aumenta la complejidad y, con ella, la superficie de ataque; CVE-2026-62914 aporta la prueba concreta. Quienes hayan leído el [artículo de julio](/blog/exchange-security-updates-juli-2026) también recordarán que la mitigación CVE-2026-42897 de mayo ya había dejado inoperativo OWA Light de forma incidental. Por tanto, la interfaz ya estaba a punto de desaparecer.

### Quienes no puedan aplicar el parche: desactivar OWA Light manualmente

Importante para todos aquellos que todavía no puedan instalar la SU de agosto, por ejemplo, porque no disponen de la habilitación ESU: Microsoft recomienda expresamente **desactivar OWA Light manualmente** en ese caso para mitigar CVE-2026-62914. Esto se realiza mediante la directiva de buzones de OWA y la página de inicio de sesión:

```powershell
Get-OwaMailboxPolicy | Set-OwaMailboxPolicy -OWALightEnabled $false
Get-OwaVirtualDirectory | Set-OwaVirtualDirectory -LogonPageLightSelectionEnabled $false
```

El primer comando desactiva la versión Light para todos los buzones de la directiva correspondiente; el segundo elimina de la página de inicio de sesión de OWA la opción «Usar la versión Light». Los cambios en el directorio virtual de OWA solo se aplican de forma fiable tras reciclar el grupo de aplicaciones de OWA o realizar un `iisreset`.

### Qué deberían comprobar ahora los administradores

La desactivación es técnicamente trivial, pero no siempre lo es desde el punto de vista organizativo: OWA Light era el discreto salvavidas para escenarios de nicho. Ahora conviene revisar los marcadores y las instrucciones del helpdesk que tengan `?layout=light` codificado de forma fija, dispositivos de quiosco y terminales con navegadores antiguos, así como las guías internas para usuarios que utilizaban la versión Light por motivos de accesibilidad. El Outlook on the web moderno funciona en todos los navegadores actuales e incluye sus propias funciones de accesibilidad; pero si no se informa previamente a los usuarios afectados, se generarán tickets.

## Por qué ahora se publica una SU cada mes y qué ocurre con Exchange SE CU1

Dos días después del lanzamiento, el equipo de Exchange respondió en una publicación de blog notablemente abierta («Where is Exchange SE CU1 anyway?») a la pregunta que muchos administradores se plantean. La versión corta: Microsoft está utilizando herramientas de IA en toda la empresa para encontrar vulnerabilidades en sus propios productos. Los equipos, incluido Exchange, están procesando actualmente los hallazgos notificados: validarlos, reproducirlos, corregirlos, probar regresiones y distribuirlos mensualmente. Desde mayo de 2026 se ha publicado así una SU de Exchange cada mes, y Microsoft afirma expresamente que este ritmo acelerado continuará.

El esperado **CU1 para Exchange SE** se retrasa precisamente por este motivo. Anunciado originalmente para la primera mitad de 2026 y aplazado después a la segunda, ya no tiene fecha objetivo. Microsoft solo quiere publicar CU1 cuando haya un mes sin una entrega urgente de seguridad entre medias; un CU superado inmediatamente por una SU supondría trabajo de actualización duplicado para muchas organizaciones. Hasta entonces, la carga de seguridad mensual se incorpora continuamente a la compilación interna de CU1.

Para la práctica, esto significa dos cosas. En primer lugar: esperar a CU1 no es una estrategia, ni para la migración a SE ni para instalar las SU. En segundo lugar: una **ventana de mantenimiento mensual** para Exchange pasa a formar parte fija del calendario operativo, tal y como hace tiempo que es habitual con los servidores Windows.

## Instalación y tareas posteriores

El procedimiento sigue siendo el habitual: primero, inventariar con el [Exchange Health Checker](https://aka.ms/ExchangeHealthChecker) qué servidores están en qué nivel de CU/SU y si quedan pasos manuales pendientes. Después, instalar la SU (si el nivel de CU está desactualizado, el [Exchange Update Wizard](https://aka.ms/ExchangeUpdateWizard) muestra la ruta), reiniciar el servidor y comprobar que todos los servicios de Exchange se hayan iniciado correctamente. Si hay servicios en estado *desactivado*, la instalación se interrumpió; entonces ayuda la solución documentada en el artículo de soporte de Microsoft sobre el error de versión de archivo o el [script SetupAssist](https://aka.ms/ExSetupAssist). Para terminar, vuelva a ejecutar el Health Checker.

Las SU son acumulativas: quien se haya saltado la SU de julio instala directamente la de agosto. Y para los entornos híbridos se aplica el conocido añadido: si se cambia el certificado de autenticación después de instalar la SU, debería ejecutarse de nuevo el Hybrid Configuration Wizard.

Una tarea posterior de julio sigue vigente: quienes todavía tengan activa la mitigación CVE-2026-42897 (M2.1.0) deberían eliminarla ahora; cómo hacerlo correctamente se explica en el [artículo sobre la SU de julio](/blog/exchange-security-updates-juli-2026).

## Procedimiento recomendado

En resumen: instale cuanto antes la SU de agosto en todos los servidores Exchange y las máquinas con Management Tools; la vulnerabilidad de Pwn2Own y la RCE con 8.8 son motivo suficiente para no esperar al próximo Patch Tuesday. Quienes no puedan aplicar el parche de inmediato deben desactivar manualmente OWA Light como medida inmediata contra CVE-2026-62914. Antes de desactivar OWA Light, identifique e informe a los grupos de usuarios afectados (marcadores antiguos, navegadores de quiosco, flujos de trabajo de accesibilidad). Después, ejecute el Health Checker, complete las tareas pendientes de julio y planifique una ventana mensual de mantenimiento de Exchange, porque el ritmo continuará.

## Fuentes

1.  [Released: August 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-august-2026-exchange-server-security-updates/4543951): Anuncio oficial de lanzamiento con las versiones compatibles, aviso sobre OWA Light, problemas conocidos y preguntas frecuentes; los comentarios incluyen las confirmaciones de la corrección de Pwn2Own (CVE-2026-62911) y de que el SettingOverride para los mensajes wrapper sigue vigente.

2.  [Upcoming retirement of OWA Light in Exchange Server – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/upcoming-retirement-of-owa-light-in-exchange-server/4534943): El anuncio previo de la desactivación, incluida la recomendación de Microsoft de desactivar OWA Light manualmente si no se aplica el parche.

3.  [Where is Exchange SE CU1 anyway? – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/where-is-exchange-se-cu1-anyway/4546837): El equipo de Exchange sobre la búsqueda de vulnerabilidades asistida por IA, el ritmo mensual continuado de las SU y el retraso de CU1.

4.  [Exchange Server build numbers and release dates – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/new-features/build-numbers-and-release-dates): Referencia para los números de compilación de las SU de agosto.

5.  [Announcing Period 2 Exchange 2016/2019 Extended Security Update (ESU) program – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/announcing-period-2-exchange-20162019-extended-security-update-esu-program/4511603): Condiciones y duración, de mayo a octubre de 2026, del programa ESU.

6.  [Wrapper messages appear in shared mailbox inbox in hybrid environments – Microsoft Support](https://support.microsoft.com/en-us/servicing/exchange/server/hotfix/2026/5105719): El problema híbrido conocido desde junio, incluida la solución temporal mediante SettingOverride.

7.  [Neue Sicherheitsupdates für Exchange Server (August 2026) – Frankys Web](https://www.frankysweb.de/neue-sicherheitsupdates-fuer-exchange-server-august-2026/): Desglose en alemán de las siete CVE con valores CVSS y compilaciones.

8.  [Set-OwaMailboxPolicy (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-owamailboxpolicy): El parámetro `OWALightEnabled` para desactivar manualmente la versión Light.

9.  [Exchange Server Health Checker – Microsoft CSS-Exchange](https://aka.ms/ExchangeHealthChecker): Inventario de niveles de CU/SU y pasos manuales pendientes antes y después de la instalación.
