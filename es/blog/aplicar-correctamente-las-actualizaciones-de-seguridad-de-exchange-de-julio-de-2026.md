---
title: "Aplicar correctamente las actualizaciones de seguridad de Exchange de julio de 2026"
navTitle: "Exchange SU 07/2026"
description: "Tras la instalación, son necesarias dos tareas de limpieza: eliminar de forma controlada la mitigación antigua de CVE-2026-42897 y revisar los grupos heredados con privilegios excesivos en Active Directory."
date: "2026-07-14"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min de lectura"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
  - active-directory-entra
slug: "aplicar-correctamente-las-actualizaciones-de-seguridad-de-exchange-de-julio-de-2026"
translationOf: "exchange-security-updates-juli-2026"
url: "https://rafaelpfister.ch/es/blog/aplicar-correctamente-las-actualizaciones-de-seguridad-de-exchange-de-julio-de-2026"
translationId: article-731b5b840aee096c
translationReview: automatic
translationSourceHash: c4f0a68a6d0b88997bcc5dadd9f5c2423dcb61c7986e179a099460335042a23a
translatedAt: 2026-07-29T12:29:38.949Z
---

# Aplicar correctamente las actualizaciones de seguridad de Exchange de julio de 2026

Con la instalación de las actualizaciones de seguridad de Exchange del 14 de julio de 2026, el trabajo aún no ha terminado. Después, los administradores deberían eliminar dos lastres heredados: la mitigación para **CVE-2026-42897** activada en mayo y dos grupos de seguridad históricos de Exchange con amplios permisos en Active Directory.

Ambas tareas son fáciles de pasar por alto. La mitigación se mantiene deliberadamente hasta que se elimine de forma controlada. Por su parte, los grupos pueden haber sobrevivido inadvertidamente a cualquier migración durante muchos años.

## Para qué versiones de Exchange está disponible la actualización

Las SU están disponibles para las siguientes versiones:

- **Exchange Server Subscription Edition (SE) RTM**: como actualización pública disponible regularmente.
- **Exchange Server 2019 CU14 y CU15**: solo para organizaciones inscritas en el **programa ESU Period 2**.
- **Exchange Server 2016 CU23**: también solo a través de ESU Period 2.

Exchange 2016 y 2019 ya no tienen soporte. Quienes no estén en el programa ESU Period 2 (vigente de mayo a octubre de 2026) ya no recibirán estas actualizaciones y no deberían seguir aplazando la migración a Exchange SE. Los entornos de Exchange Online ya están protegidos; en configuraciones híbridas, no obstante, la SU debe instalarse en todos los servidores de Exchange, incluidos los servidores dedicados exclusivamente a administración. Como es habitual, las CVE concretas abordadas se indican en la Security Update Guide (filtro «Server Software» para Exchange SE o «ESU» para 2016/2019).

Hay un problema conocido en la versión actual: en entornos híbridos, pueden aparecer los denominados *mensajes contenedor* en la bandeja de entrada de buzones compartidos. Consulte los detalles en el artículo de soporte de Microsoft correspondiente.

## Eliminar la mitigación de CVE-2026-42897 después de la instalación

### Breve repaso

CVE-2026-42897 se anunció el 14 de mayo de 2026: una vulnerabilidad de cross-site scripting (suplantación) en Outlook Web Access. Un atacante envía un correo electrónico especialmente manipulado; si la víctima lo abre en OWA y se cumplen determinadas condiciones de interacción, puede ejecutarse JavaScript arbitrario en el contexto del navegador. Exchange 2016, 2019 y SE se veían afectados en *cualquier* nivel de parche. Microsoft publicó ese mismo día una mitigación de emergencia (ID **M2.1.x**, la regla concreta de IIS se llama **M2.1.0**) y entregó la corrección propiamente dicha con la SU de junio de 2026.

### Por qué la actualización de julio *no* elimina automáticamente la mitigación

Este es el punto que más sorprende: incluso después de instalar la SU de julio, una mitigación ya aplicada permanece activa. El motivo está en el mecanismo. La mitigación es una **regla de reescritura de URL de IIS basada en Content Security Policy**, aplicada *fuera* del instalador MSI, ya sea mediante el Emergency Mitigation Service (servicio EM) o mediante el script EOMT. El parche MSI sustituye binarios, pero no administra estas reglas de IIS establecidas fuera de banda. Por ello, eliminarlas es un paso manual independiente.

Por cierto: la mitigación nunca protegió a los clientes IE ni a Edge en modo IE, porque Internet Explorer no admite CSP. Quienes utilizaban esos clientes nunca estuvieron protegidos solo con la mitigación. Es otro argumento para aplicar el parche pronto en vez de confiar en la mitigación.

### La trampa: el servicio EM vuelve a aplicar la mitigación

Quien elimine la regla prematuramente se llevará una sorpresa. El servicio EM se ejecuta cada hora y compara el estado actual con las directrices proporcionadas por el Office Config Service (Flighting). La asignación de «qué compilación necesita qué mitigación» se gestiona en el servidor. Solo un cambio en el servidor marca la compilación de julio de 2026 como «mitigación ya no necesaria». Según Microsoft, este cambio se implementó por completo alrededor del 16 de julio de 2026. Hasta entonces, el servicio EM simplemente volverá a crear una regla M2.1.0 eliminada en la siguiente ejecución horaria.

En la práctica, esto significa: o se espera para eliminarla manualmente hasta después del 16 de julio, o se bloquea explícitamente la mitigación para que no se reactive.

### Cómo eliminar correctamente la mitigación (ruta del servicio EM)

Primero, compruebe qué se ha aplicado realmente:

```powershell
Get-ExchangeServer -Identity <NombreDelServidor> | Format-List Name,MitigationsApplied,MitigationsBlocked
```

Para evitar la reactivación, se añade el ID de la mitigación a la lista de bloqueados: el servicio EM ignora las entradas allí incluidas durante la ejecución horaria.

```powershell
Set-ExchangeServer -Identity <NombreDelServidor> -MitigationsBlocked @("M2.1.0")
```

A continuación, elimine la regla de IIS propiamente dicha. Conviene saber, y rara vez se documenta, que el servicio EM crea sus reglas de reescritura de URL con el **prefijo «EEMS `<Mitigation-ID>` `<Beschreibung>`»**. Así se pueden identificar claramente en el Administrador de IIS, bajo URL Rewrite (o mediante `appcmd`/PowerShell en `applicationHost.config`), sin tener que adivinar qué regla corresponde a la mitigación. Tras implementar el cambio en el servidor, puede quitar de nuevo el bloqueo (`-MitigationsBlocked @()`) si solo lo estableció como solución temporal.

### Ruta EOMT (entornos aislados o sin conexión)

Si la mitigación se aplicó mediante el **script EOMT** descargable (https://aka.ms/UnifiedEOMT), se revierte mediante el interruptor de rollback:

```powershell
.\EOMT.ps1 -RollbackMitigation -CVE "CVE-2026-42897"
```

Aquí también hay un detalle poco conocido: antes de cada cambio, EOMT guarda el estado inicial de IIS en un **archivo de copia de seguridad JSON específico de la CVE** en `%WINDIR%\System32\inetsrv\config\`. El rollback lee exactamente ese archivo y restaura la configuración original. Importante: una mitigación aplicada con un script heredado (EOMTv2, etc.) también debe revertirse con su propio mecanismo de rollback: los formatos de copia de seguridad no son compatibles.

### Por qué merece la pena eliminarla

La mitigación no es «gratuita». Mientras esté activa, mantiene sus efectos secundarios conocidos: la función de OWA «Imprimir calendario» no funciona, es posible que las imágenes insertadas no se muestren correctamente en el panel de lectura de OWA, OWA Light (`/?layout=light`) está averiado (de todos modos se desactivará próximamente) y los calendarios publicados a veces devuelven el error 500. Especialmente problemático para la monitorización: el conjunto de estado **OWACalendar.Proxy** puede pasar a *unhealthy* y provocar falsas alarmas en la supervisión. Quien haya instalado la SU pero mantenga la mitigación acabará persiguiendo fantasmas. Una vez instalada la actualización *y* eliminada la mitigación, estos problemas conocidos también desaparecen.

Un caso especial: en entornos mixtos, los servidores que aún no se hayan actualizado pueden conservar la mitigación. Sin embargo, debe tenerse en cuenta que la integración de Office Online Server (OOS) puede no volver a funcionar correctamente hasta que *todos* los servidores de Exchange de la organización estén en la versión de julio.

## Health Checker: detectar grupos de seguridad muy antiguos

El segundo punto, independiente de la versión de la SU: el **Exchange Health Checker** (https://aka.ms/ExchangeHealthChecker) ahora comprueba la existencia de dos grupos de seguridad desaconsejados desde hace tiempo: **«Exchange Domain Servers»** y **«Exchange Enterprise Servers»**.

### De dónde proceden estos grupos y por qué son un riesgo

Estos dos grupos proceden del modelo de permisos de Exchange 2000/2003 y están desaconsejados desde Exchange 2007. Con Exchange 2007/2010 llegó el modelo Split Permissions o RBAC, y desde entonces simplemente ya no se utilizan. El problema es que no han desaparecido por ello. En muchos directorios llevan unos dos decenios sin recibir atención y aún conservan en parte ACL amplias del modelo antiguo, es decir, más permisos de los que tendría jamás un grupo de seguridad moderno de Exchange.

Eso es precisamente lo que los convierte en un vector de ataque. Un grupo inactivo con permisos amplios y permanentes constituye una cadena de escalada clásica: quien logre añadirse a sí mismo (o a una cuenta controlada) a uno de estos grupos heredará sus permisos en el directorio. Como nadie supervisa activamente el grupo, una manipulación así apenas se detecta.

### Por qué la mayoría de administradores no los tienen presentes

Estos grupos son un punto ciego por varios motivos: llevan inactivos unos 20 años, por lo general ya existían antes de que el equipo actual asumiera sus funciones, sobreviven sin problemas a cualquier migración y hasta ahora Health Checker nunca los mostraba. Especialmente delicado: sobreviven incluso a la retirada *completa* de Exchange local. Quien haya eliminado el último servidor de Exchange normalmente limpia los objetos de servidor, pero pasa por alto por completo estos grupos heredados.

### Limpieza

Health Checker informará de estos grupos automáticamente en el futuro. Manualmente se pueden localizar en Active Directory (normalmente en el contenedor `Users`) o mediante PowerShell:

```powershell
Get-ADGroup -Filter "Name -eq 'Exchange Domain Servers' -or Name -eq 'Exchange Enterprise Servers'"
```

Procedimiento: comprobar la pertenencia y las posibles referencias de ACL personalizadas, asegurarse de que nada de producción dependa de ellos y, a continuación, eliminar los grupos. Dado que están desaconsejados desde 2007, se pueden eliminar sin riesgo en la gran mayoría de entornos. Quienes ya no operen ningún Exchange local deberían planificar al mismo tiempo una limpieza más amplia de AD conforme a las instrucciones oficiales de Microsoft.

Hayes Jupe ha publicado una guía detallada para eliminar los grupos en su entrada de blog [Latest Exchange health check script and deprecated groups](https://www.hayesjupe.com/latest-exchange-health-check-script-and-deprecated-groups/).

## Procedimiento recomendado

En resumen, este es el proceso práctico: primero, inventariar el entorno con Health Checker (muestra CU/SU faltantes, pasos manuales pendientes *y*, ahora, también los grupos heredados). Después, instalar la CU actual y la SU de julio, reiniciar el servidor y comprobar que todos los servicios de Exchange se hayan iniciado correctamente. A continuación, ejecutar de nuevo Health Checker, eliminar la mitigación de CVE-2026-42897 (después del 16 de julio o bloqueando previamente el ID M2.1.0) y, por último, limpiar los grupos de seguridad desaconsejados. Las SU son acumulativas: quien se encuentre en una CU compatible no necesita instalar todas las SU intermedias, sino que puede instalar directamente la más reciente.

## Fuentes

1.  [Released: July 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-july-2026-exchange-server-security-updates/4534146): Anuncio oficial de la versión de julio con las versiones compatibles y el problema conocido de los mensajes contenedor.

2.  [Addressing Exchange Server May 2026 vulnerability CVE-2026-42897 – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/addressing-exchange-server-may-2026-vulnerability-cve-2026-42897/4518498): Aviso de seguridad original, incluida la mitigación de emergencia y los efectos secundarios conocidos en OWA.

3.  [Released: June 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-june-2026-exchange-server-security-updates/4524491): La versión de junio que entregó la corrección propiamente dicha para CVE-2026-42897.

4.  [Exchange Emergency Mitigation Service (Exchange EM Service) – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-emergency-mitigation-service): Funcionamiento del servicio EM, que compara las mitigaciones cada hora y vuelve a crear una regla eliminada prematuramente.

5.  [Set-ExchangeServer (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-exchangeserver): Parámetros `MitigationsApplied` y `MitigationsBlocked`, para comprobar mitigaciones e impedir su reactivación.

6.  [Exchange On-premises Mitigation Tool (EOMT) – Microsoft CSS-Exchange](https://microsoft.github.io/CSS-Exchange/Security/EOMT/): El script EOMT, incluido el interruptor de rollback y la copia de seguridad JSON específica de la CVE del estado inicial de IIS.

7.  [CVE-2026-42897 Detail – NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-42897): Descripción técnica y evaluación de la vulnerabilidad en la National Vulnerability Database.
