---
title: "Realizar correctamente las tareas posteriores a las actualizaciones de seguridad de Exchange de julio de 2026"
navTitle: "Exchange SU 07/2026"
description: "Tras la instalación, se requieren dos tareas de limpieza: retirar de forma controlada la mitigación antigua de CVE-2026-42897 y revisar los grupos heredados con privilegios excesivos en Active Directory."
date: "2026-07-14"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "6 min de lectura"
themen:
  - exchange-updates
  - active-directory-entra
slug: "aplicar-correctamente-las-actualizaciones-de-seguridad-de-exchange-de-julio-de-2026"
translationOf: "exchange-security-updates-juli-2026"
translationId: article-731b5b840aee096c
translationReview: automatic
translationSourceHash: e5d9295515965d3e7801752cd605f6d2a78cacfc9fb965e0f63d645658b39e9b
translatedAt: 2026-09-05T07:52:34.748Z
url: https://rafaelpfister.ch/es/blog/aplicar-correctamente-las-actualizaciones-de-seguridad-de-exchange-de-julio-de-2026
translationModel: gpt-5.6-terra
---

# Realizar correctamente las tareas posteriores a las actualizaciones de seguridad de Exchange de julio de 2026

Con la instalación de las actualizaciones de seguridad de Exchange del 14 de julio de 2026, el trabajo aún no ha terminado. Después, los administradores deberían eliminar dos legados: la mitigación activada en mayo para **CVE-2026-42897** y dos grupos de seguridad históricos de Exchange con amplios permisos en Active Directory.

Ambas tareas son fáciles de pasar por alto. La mitigación permanece deliberadamente activa hasta que se elimina de forma controlada. Por su parte, los grupos pueden haber sobrevivido inadvertidamente a cualquier migración durante muchos años.

## Para qué versiones de Exchange está disponible la actualización

Las SU están disponibles para las siguientes versiones:

- **Exchange Server Subscription Edition (SE) RTM**: como actualización pública disponible de forma general.
- **Exchange Server 2019 CU14 y CU15**: solo para organizaciones inscritas en el **programa ESU del período 2**.
- **Exchange Server 2016 CU23**: también solo mediante ESU del período 2.

Exchange 2016 y 2019 están fuera de soporte. Quienes no estén en el programa ESU del período 2 (válido de mayo a octubre de 2026) ya no recibirán estas actualizaciones y no deberían seguir posponiendo la migración a Exchange SE. Los entornos de Exchange Online ya están protegidos; en configuraciones híbridas, la SU debe instalarse igualmente en todos los servidores Exchange, incluidos los servidores dedicados únicamente a administración. Los CVE concretos abordados se indican, como de costumbre, en la Security Update Guide (filtro «Server Software» para Exchange SE o «ESU» para 2016/2019).

Existe un problema conocido en la versión actual: en entornos híbridos pueden aparecer los llamados *mensajes wrapper* en la bandeja de entrada de buzones compartidos. Los detalles se encuentran en el artículo de soporte de Microsoft correspondiente.

## Eliminar la mitigación de CVE-2026-42897 tras la instalación

### Breve repaso

CVE-2026-42897 se anunció el 14 de mayo de 2026: una vulnerabilidad de cross-site scripting (suplantación) en Outlook Web Access. Un atacante envía un correo electrónico especialmente manipulado; si la víctima lo abre en OWA y se cumplen determinadas condiciones de interacción, es posible ejecutar JavaScript arbitrario en el contexto del navegador. Se veían afectados Exchange 2016, 2019 y SE en *cualquier* nivel de parche. Microsoft publicó el mismo día una mitigación de emergencia (ID **M2.1.x**, cuya regla IIS concreta se denomina **M2.1.0**) y proporcionó la corrección propiamente dicha con la SU de junio de 2026.

### Por qué la actualización de julio *no* elimina automáticamente la mitigación

Este es el punto que más sorprende: incluso después de instalar la SU de julio, una mitigación ya aplicada permanece activa. El motivo está en el mecanismo. La mitigación es una **regla de reescritura de URL de IIS basada en Content Security Policy**, aplicada *fuera* del instalador MSI, ya sea mediante Exchange Emergency Mitigation Service (EM Service) o mediante el script EOMT. El parche MSI sustituye archivos binarios, pero no administra estas reglas IIS establecidas fuera de banda. Por ello, su eliminación es un paso manual independiente.

Por cierto: la mitigación nunca protegió a los clientes de IE ni a Edge en modo IE, ya que Internet Explorer no admite CSP. Quienes utilizaban tales clientes nunca estuvieron protegidos únicamente por la mitigación. Este es otro argumento para aplicar parches cuanto antes en lugar de confiar en la mitigación.

### El punto delicado: el EM Service vuelve a aplicar la mitigación

Una regla eliminada prematuramente no permanece eliminada de forma permanente. El EM Service se ejecuta cada hora y compara el estado actual con las directrices proporcionadas por el Office-Config-Service (Flighting). La asignación de «qué compilación necesita qué mitigación» se gestiona en el servidor. Solo un cambio del lado del servidor marca la compilación de julio de 2026 como «mitigación ya no necesaria». Según Microsoft, este cambio no se desplegó por completo hasta alrededor del 16 de julio de 2026. Hasta entonces, el EM Service simplemente volverá a añadir una regla M2.1.0 eliminada en la siguiente ejecución horaria.

En la práctica, esto significa: o bien se espera para eliminarla manualmente hasta después del 16 de julio, o se bloquea explícitamente la mitigación para evitar que se reactive.

### Cómo eliminar correctamente la mitigación (ruta del EM Service)

Primero, compruebe qué se ha aplicado:

```powershell
Get-ExchangeServer -Identity <Servername> | Format-List Name,MitigationsApplied,MitigationsBlocked
```

Para evitar la reactivación, se añade el ID de mitigación a la lista de bloqueo: el EM Service ignora las entradas presentes allí durante su ejecución horaria.

```powershell
Set-ExchangeServer -Identity <Servername> -MitigationsBlocked @("M2.1.0")
```

A continuación, elimine la regla IIS propiamente dicha. Es útil saber, y está poco documentado, que el EM Service crea sus reglas de reescritura de URL con el **prefijo «EEMS `<Mitigation-ID>` `<Beschreibung>`»**. Así se pueden localizar claramente en IIS Manager, en URL Rewrite (o mediante `appcmd`/PowerShell en la `applicationHost.config`), sin tener que adivinar qué regla pertenece a la mitigación. Tras el despliegue del cambio del lado del servidor, se puede retirar de nuevo el bloqueo (`-MitigationsBlocked @()`), siempre que se hubiera establecido solo como solución temporal.

### Ruta EOMT (entornos aislados o air-gapped)

Si la mitigación se aplicó mediante el **script EOMT** descargable (https://aka.ms/UnifiedEOMT), la reversión se realiza con el modificador de rollback:

```powershell
.\EOMT.ps1 -RollbackMitigation -CVE "CVE-2026-42897"
```

También aquí hay un detalle poco conocido: antes de cada cambio, EOMT guarda el estado inicial de IIS en un **archivo de copia de seguridad JSON específico del CVE** en `%WINDIR%\System32\inetsrv\config\`. El rollback lee exactamente este archivo y restaura la configuración original. Importante: una mitigación aplicada con un script heredado (EOMTv2, etc.) también debe revertirse mediante su propio mecanismo de rollback: los formatos de copia de seguridad no son compatibles.

### Por qué merece la pena eliminarla

La mitigación no es «gratuita». Mientras esté activa, arrastra sus efectos secundarios conocidos: la función de OWA «Imprimir calendario» no funciona, es posible que las imágenes en línea no se muestren correctamente en el panel de lectura de OWA, OWA Light (`/?layout=light`) está defectuoso (de todos modos se desactivará próximamente) y los calendarios publicados devuelven en parte errores 500. Especialmente engañoso para la supervisión: el healthset **OWACalendar.Proxy** puede pasar a *unhealthy* y provocar así falsas alarmas en el monitoreo. Quien haya instalado la SU pero deje activa la mitigación acabará buscando errores que no existen. En cuanto se instale la actualización *y* se elimine la mitigación, estos problemas conocidos también desaparecerán.

Un caso especial: en entornos mixtos, los servidores aún no actualizados pueden conservar la mitigación. Sin embargo, conviene saber que la integración de Office Online Server (OOS) puede volver a funcionar correctamente solo cuando *todos* los servidores Exchange de la organización estén en el nivel de julio.

## Health Checker: detectar grupos de seguridad muy antiguos

El segundo punto, independiente de la versión de la SU: el **Exchange Health Checker** (https://aka.ms/ExchangeHealthChecker) ahora comprueba la existencia de dos grupos de seguridad obsoletos desde hace mucho tiempo: **«Exchange Domain Servers»** y **«Exchange Enterprise Servers»**.

### De dónde proceden estos grupos y por qué son un riesgo

Estos dos grupos proceden del modelo de permisos de Exchange 2000/2003 y están obsoletos desde Exchange 2007. Con Exchange 2007/2010 llegó el modelo de permisos divididos y RBAC, y desde entonces simplemente ya no se utilizan. El problema es que no han desaparecido. En muchos directorios llevan alrededor de dos décadas sin que nadie les preste atención y, en algunos casos, aún poseen ACL amplias del modelo antiguo, es decir, más permisos de los que tendría un grupo de seguridad moderno de Exchange.

Precisamente eso los convierte en un vector de ataque. Un grupo inactivo con permisos amplios persistentes constituye una cadena de escalada clásica: quien consiga añadirse a sí mismo (o a una cuenta controlada) a dicho grupo hereda sus permisos en el directorio. Como nadie supervisa activamente el grupo, una manipulación de este tipo apenas se detecta.

### Por qué la mayoría de administradores no los conoce

Estos grupos son un punto ciego por varios motivos: llevan inactivos unos 20 años, normalmente ya existían antes de que asumiera el equipo actual, sobreviven sin problemas a cualquier migración y hasta ahora Health Checker nunca los mostraba. Especialmente delicado: incluso sobreviven a la retirada *completa* de Exchange on-premises. Quien ha eliminado el último servidor Exchange suele limpiar los objetos de servidor, pero pasa por alto por completo estos grupos heredados.

### Limpieza

En el futuro, Health Checker informará automáticamente de los grupos. También se pueden encontrar manualmente en Active Directory (normalmente en el contenedor `Users`) o mediante PowerShell:

```powershell
Get-ADGroup -Filter "Name -eq 'Exchange Domain Servers' -or Name -eq 'Exchange Enterprise Servers'"
```

Procedimiento: compruebe la pertenencia y las posibles referencias ACL personalizadas, asegúrese de que nada productivo dependa de ellas y, a continuación, elimine los grupos. Dado que están obsoletos desde 2007, pueden eliminarse sin riesgo en la gran mayoría de los entornos. Quienes ya no operen ningún Exchange on-premises deberían aprovechar para planificar una limpieza de AD más exhaustiva conforme a las instrucciones oficiales de Microsoft.

Hayes Jupe ha publicado en su blog una guía detallada para eliminar los grupos: [Latest Exchange health check script and deprecated groups](https://www.hayesjupe.com/latest-exchange-health-check-script-and-deprecated-groups/).

## Procedimiento recomendado

En resumen, el flujo de trabajo práctico es el siguiente: primero, inventariar el entorno con Health Checker (muestra CU/SU faltantes, pasos manuales pendientes *y* ahora también los grupos heredados). A continuación, instalar la CU actual y la SU de julio, reiniciar el servidor y comprobar que todos los servicios Exchange se hayan iniciado correctamente. Después, ejecutar de nuevo Health Checker, eliminar la mitigación de CVE-2026-42897 (después del 16 de julio o bloqueando previamente el ID M2.1.0) y, por último, limpiar los grupos de seguridad obsoletos. Las SU son acumulativas: quien esté en una CU compatible no tiene que instalar todas las SU intermedias, sino que instala directamente la más reciente.

## Fuentes

1.  [Released: July 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-july-2026-exchange-server-security-updates/4534146): Anuncio oficial de la versión de julio con las versiones compatibles y el problema conocido de los mensajes wrapper.

2.  [Addressing Exchange Server May 2026 vulnerability CVE-2026-42897 – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/addressing-exchange-server-may-2026-vulnerability-cve-2026-42897/4518498): Aviso de seguridad original, incluida la mitigación de emergencia y los efectos secundarios conocidos en OWA.

3.  [Released: June 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-june-2026-exchange-server-security-updates/4524491): La versión de junio, que proporcionó la corrección propiamente dicha para CVE-2026-42897.

4.  [Exchange Emergency Mitigation Service (Exchange EM Service) – Microsoft Learn](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-emergency-mitigation-service): Funcionamiento del EM Service, que compara las mitigaciones cada hora y vuelve a añadir una regla eliminada prematuramente.

5.  [Set-ExchangeServer (ExchangePowerShell) – Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-exchangeserver): Parámetros `MitigationsApplied` y `MitigationsBlocked` para comprobar mitigaciones e impedir su reactivación.

6.  [Exchange On-premises Mitigation Tool (EOMT) – Microsoft CSS-Exchange](https://microsoft.github.io/CSS-Exchange/Security/EOMT/): El script EOMT, incluido el modificador de rollback y la copia de seguridad JSON específica del CVE del estado inicial de IIS.

7.  [CVE-2026-42897 Detail – NVD](https://nvd.nist.gov/vuln/detail/CVE-2026-42897): Descripción técnica y evaluación de la vulnerabilidad en la National Vulnerability Database.
