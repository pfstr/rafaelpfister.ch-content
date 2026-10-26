---
title: "Microsoft Entra Connect Sync 2.6.91.0 y 2.6.92.0: correcciones de seguridad, migración a Cloud Sync y actualización obligatoria antes de abril de 2027"
navTitle: "Microsoft Entra Connect Sync 2.6.92.0"
description: "La versión 2.6.91.0 incorpora correcciones de seguridad, un cambio guiado a Cloud Sync y soluciona los problemas conocidos de la 2.6.84.0; el hotfix 2.6.92.0 corrige la configuración de la autenticación de paso a través. Antes del 7 de abril de 2027, todos los servidores deben ejecutar al menos la versión 2.6.84.0 con autenticación basada en aplicaciones."
date: "2026-10-26"
kategorie: "Microsoft Entra"
timeToRead: "8 min de lectura"
themen:
  - microsoft-entra
  - active-directory-entra
produkte:
  - "entra"
  - "entra-connect-releases"
protokolle:
  - "releases"
slug: "microsoft-entra-connect-sync-2-6-91-0-y-2-6-92-0-correcciones-de-seguridad-migracion-a-cloud"
translationId: article-59a51ef69f2e3538
translationOf: entra-connect-2-6-92-0
url: https://rafaelpfister.ch/es/blog/microsoft-entra-connect-sync-2-6-91-0-y-2-6-92-0-correcciones-de-seguridad-migracion-a-cloud
translationSourceHash: ff045b1bab72cea3839167d59c0a71010e80a3a20b1644acef89c79921f6f219
translationModel: gpt-5.6-terra
translatedAt: 2026-10-26T08:25:23.380Z
translationReview: automatic
---

# Microsoft Entra Connect Sync 2.6.91.0 y 2.6.92.0: correcciones de seguridad, migración a Cloud Sync y actualización obligatoria antes de abril de 2027

Microsoft publicó Entra Connect Sync 2.6.91.0 el 16 de septiembre de 2026 y el hotfix 2.6.92.0 el 23 de septiembre de 2026, ambos solo como descarga a través del Centro de administración de Microsoft Entra. La versión 2.6.91.0 corrige vulnerabilidades de seguridad en componentes de terceros incluidos, pone a disposición general el inicio de sesión resistente al phishing en el asistente de configuración, incorpora un proceso de migración guiado a Microsoft Entra Cloud Sync y corrige los tres problemas conocidos de la 2.6.84.0. La versión 2.6.92.0 corrige un error de la 2.6.91.0 al configurar la autenticación de paso a través. Al mismo tiempo, Microsoft establece un plazo fijo: antes del 7 de abril de 2027, todos los servidores deben ejecutar como mínimo la versión 2.6.84.0 y utilizar autenticación basada en aplicaciones; de lo contrario, la sincronización se detendrá. El artículo está dirigido a administradores que operan Entra Connect Sync. El estado anterior se describe en el artículo sobre [Entra Connect Sync 2.6.84.0](/blog/entra-connect-2-6-84-0).

## Versiones de un vistazo

| Versión | Fecha | Tipo | Puntos clave |
|---|---|---|---|
| 2.6.91.0 | 16.09.2026 | Versión de funcionalidades y seguridad (descarga) | Correcciones de seguridad en componentes de terceros, asistente de migración a Cloud Sync, disponibilidad general del inicio de sesión con passkey en el asistente, nuevos permisos de Microsoft Graph, correcciones de los problemas conocidos de la 2.6.84.0 |
| 2.6.92.0 | 23.09.2026 | Hotfix con correcciones de seguridad (descarga) | Corrige el registro fallido del Authentication Agent al activar la autenticación de paso a través |

Microsoft clasifica la versión 2.6.92.0 en las notas de la versión como una versión con correcciones de seguridad y recomienda actualizar «lo antes posible». No se ha anunciado un despliegue de actualización automática para ninguna de las dos versiones (a fecha de 26 de octubre de 2026). La versión 2.6.84.0 también sigue estando disponible únicamente como descarga.

## Actualización obligatoria antes del 7 de abril de 2027

El historial de versiones y la guía de actualización incluyen ahora un aviso idéntico: Entra Connect Sync debe actualizarse a la versión 2.6.84.0 o posterior y cambiarse a la autenticación basada en aplicaciones (ABA) antes del 7 de abril de 2027. Microsoft retirará la autenticación heredada mediante la cuenta de sincronización de directorios; los servidores que no cumplan estos requisitos dejarán de sincronizar después de esa fecha. Según Microsoft, si la sincronización se detiene, la actualización a la versión actual seguida de la configuración de ABA restablece el funcionamiento.

Para la planificación es decisivo un cambio de comportamiento introducido en la versión 2.6.84.0: desde esta versión, Entra Connect ya no cambia automáticamente los servidores existentes a ABA. Solo las instalaciones nuevas configuran ABA de forma automática. Por tanto, una actualización por sí sola no cumple el plazo. Debe cambiar los servidores existentes en el asistente mediante **Additional tasks** > **Configure application-based authentication to Microsoft Entra ID**.

El asistente muestra el método que utiliza actualmente un servidor en **View or export current configuration**. Microsoft también indica este cmdlet:

```powershell
Get-ADSyncEntraConnectorCredential
```

El valor devuelto `ConnectorIdentityType` es `ServiceAccount` (cuenta heredada) o `Application` (ABA). Si aparece `ServiceAccount`, es necesario realizar el cambio antes del 7 de abril de 2027. Tras completar el cambio correctamente, Microsoft recomienda eliminar de Entra ID la antigua cuenta de sincronización de directorios.

## Versión 2.6.91.0

### Correcciones de seguridad

Las notas de la versión mencionan vulnerabilidades de seguridad corregidas en componentes de terceros incluidos, permisos reforzados en archivos temporales locales y descargas protegidas de imágenes de perfil en el conector de SharePoint. Microsoft no indica números CVE, valores CVSS ni los componentes afectados. Por ello, no es posible evaluar la gravedad basándose en las Fuentes públicas.

### Nuevos permisos de Microsoft Graph y Conditional Access

Entra Connect recibe permisos adicionales de Microsoft Graph. Microsoft señala expresamente que quienes utilicen directivas de Conditional Access basadas en aplicaciones deberían revisar las directivas dirigidas a `Microsoft.Azure.SyncFabric` o al servicio de informes de Microsoft 365. Revise estas directivas antes de la actualización para evitar que se bloqueen los nuevos accesos.

### Migración guiada a Cloud Sync

La novedad es un proceso guiado para cambiar de Entra Connect Sync a Microsoft Entra Cloud Sync. Incluye una evaluación de la configuración existente, la configuración del Provisioning Agent, una activación escalonada y una validación. La función solo está disponible en Azure Public Cloud. La guía de actualización de Microsoft recomienda en cualquier caso comprobar antes de actualizar si Cloud Sync es adecuado para el propio entorno.

### Inicio de sesión resistente al phishing en el asistente disponible de forma general

El inicio de sesión mediante Windows Web Account Manager, introducido como vista previa en la versión 2.6.84.0, ahora está disponible de forma general y activo de manera predeterminada. El cuadro de diálogo admite passkeys, claves de seguridad FIDO2 y contraseñas, reutiliza una sesión de inicio de sesión existente para otros servicios Entra y mantiene la rotación de claves Kerberos para Seamless Single Sign-On.

### Cmdlets de PowerShell modificados

La versión 2.6.84.0 había hecho obligatorio el parámetro `-AADUsername` para `Set-ADSyncAADCompanyFeature` y `Set-ADSyncAADPasswordSyncState`. Con la versión 2.6.91.0, el parámetro vuelve a ser opcional: si falta, Entra Connect deriva una sugerencia de inicio de sesión de la configuración del conector y abre un inicio de sesión interactivo. Esto se aplica a `Set-ADSyncAADCompanyFeature`, `Set-ADSyncAADPasswordSyncState`, `Enable-ADSyncExportDeletionThreshold`, `Set-ADSyncScheduler` y `Set-ADSyncDirSyncConfiguration`. Por tanto, el inicio de sesión interactivo sigue formando parte de estos cmdlets; debería probar tras la actualización los scripts que los ejecuten de forma desatendida.

Quienes configuren Seamless Single Sign-On con el módulo independiente de PowerShell deben importar `ADSync.psd1` antes de `AzureADSSO.psd1`.

### Otros cambios

- El cuadro de diálogo **Select Containers** del Synchronization Service Manager ahora es de solo lectura. Los cambios en la selección de contenedores se realizan mediante **Customize synchronization options** en el asistente.
- Compatibilidad con otro entorno de nube soberana, incluida la autenticación de paso a través, Seamless Single Sign-On, Password Writeback y supervisión mediante Health Agent. Las notas de la versión no indican de qué entorno se trata.
- La SQL Server 2022 LocalDB incluida se actualizó de la versión 16.0.4250.1 a la 16.0.4262.2.

### Correcciones de errores

- La instalación o actualización con una base de datos ADSync existente podía fallar con el error `0xE0474352` (problema conocido de la 2.6.84.0).
- Una actualización no actualizaba una Visual C++ 2015–2022 Redistributable instalada anterior a la versión 14.41.
- El asistente podía cerrarse al iniciarse si la directiva de Windows PowerShell Transcription contenía un directorio de salida no válido.
- La rotación de certificados de la autenticación basada en aplicaciones podía eliminar la última Application Key utilizable en caso de retrasos de replicación en el directorio.
- El Synchronization Service Manager podía cerrarse al enumerar las particiones de directorio en **Connector Properties** (problema conocido de la 2.6.84.0).
- **Change Credentials** no se abría para filas de conectores fuera del área visible cuando hay muchos conectores de Active Directory configurados.
- Al expandir un dominio previamente deseleccionado en la página **Domain and OU filtering**, el asistente podía volver a seleccionar el dominio y activar la sincronización para todo el dominio al finalizar (problema conocido de la 2.6.84.0).

El último punto merece atención: quienes ejecuten la versión 2.6.84.0 y hayan vuelto a recorrer el asistente después de actualizar deberían comprobar el filtrado de dominios y unidades organizativas. Un dominio activado involuntariamente provoca objetos adicionales en Entra ID.

## Versión 2.6.92.0

El hotfix corrige un error de la versión 2.6.91.0: al activar la autenticación de paso a través mediante el asistente, podía fallar el registro del Microsoft Entra Connect Authentication Agent instalado localmente. Las notas de la versión no mencionan más cambios. Microsoft marca la versión 2.6.92.0 como una versión con correcciones de seguridad; no se enumeran vulnerabilidades concretas.

## Problemas conocidos

| Problema | Versión afectada | Estado |
|---|---|---|
| La instalación o actualización con una base de datos ADSync existente falla con `0xE0474352` | 2.6.84.0 | Corregido en la 2.6.91.0 |
| El Synchronization Service Manager se cierra en **Connector Properties** | 2.6.84.0 | Corregido en la 2.6.91.0 |
| El asistente vuelve a seleccionar un dominio deseleccionado | 2.6.84.0 | Corregido en la 2.6.91.0 |
| La activación de PTA en el asistente falla durante el registro del agente | 2.6.91.0 | Corregido en la 2.6.92.0 |
| La sincronización falla tras actualizar (`System.IO.FileLoadException`) si `miiserver.exe.config` se modificó anteriormente | 2.5.190.0, 2.6.1.0 | Sigue figurando en el historial de versiones; solución alternativa mediante bindingRedirect manual; véase el [artículo sobre la 2.6.84.0](/blog/entra-connect-2-6-84-0) |

La versión retirada 2.6.79.0 sigue sin estar disponible. Microsoft remite ahora a los afectados a la versión 2.6.92.0: desinstalar e instalar la versión actual.

## Plazos de soporte

Cada versión 2.x deja de recibir soporte doce meses después de la publicación de la versión sucesora. Con las dos nuevas versiones, se aplican estos plazos:

| Versión | Publicada | Fin del soporte |
|---|---|---|
| 2.5.79.0 | 1 de septiembre de 2025 | 23 de octubre de 2026 (finalizado) |
| 2.5.190.0 | 19 de noviembre de 2025 | 2 de febrero de 2027 |
| 2.6.1.0 | 2 de febrero de 2026 | 10 de marzo de 2027 |
| 2.6.3.0 | 10 de marzo de 2026 | 7 de julio de 2027 |
| 2.6.84.0 | 7 de julio de 2026 | 16 de septiembre de 2027 |
| 2.6.91.0 | 16 de septiembre de 2026 | 23 de septiembre de 2027 |
| 2.6.92.0 | 23 de septiembre de 2026 | pendiente |

Para las versiones 2.5.190.0 a 2.6.3.0 también se aplica el plazo del 7 de abril de 2027: están por debajo de la versión mínima requerida 2.6.84.0 y dejarán de sincronizar después de esa fecha, aunque el fin de soporte regular de la versión 2.6.3.0 no llegue hasta julio de 2027. La versión 2.5.79.0 no cuenta con soporte desde el 23 de octubre de 2026; según Microsoft, las versiones que ya no reciben soporte pueden dejar de funcionar inesperadamente.

## Procedimiento recomendado

La versión objetivo es la 2.6.92.0. Contiene las correcciones de seguridad de ambas versiones, resuelve todos los problemas conocidos de las versiones 2.6.84.0 y 2.6.91.0 y cumple la versión mínima para el plazo del 7 de abril de 2027. El instalador solo está disponible en el Centro de administración de Microsoft Entra, en **Microsoft Entra Connect | Get started**, pestaña **Manage**; los requisitos son .NET Framework 4.7.2 y TLS 1.2. Antes de actualizar, revise las directivas de Conditional Access basadas en aplicaciones para `Microsoft.Azure.SyncFabric` y el servicio de informes de Microsoft 365, así como los scripts que invoquen los cmdlets de configuración en la nube mencionados. Para actualizaciones de bajo riesgo, Microsoft recomienda la Swing Migration: primero actualizar el servidor de ensayo, verificar la configuración tras Full Import y Full Synchronization, intercambiar después los roles y actualizar el anterior servidor activo. Según Microsoft, quienes no hayan actualizado durante 12 a 18 meses deberían elegir esta vía. Después de la actualización, cambie a la autenticación basada en aplicaciones los servidores existentes que todavía informen `ServiceAccount` y compruebe el filtrado de dominios y unidades organizativas en los servidores que ejecutaban anteriormente la versión 2.6.84.0.

## Fuentes

1.  [Microsoft Entra Connect: Version release history](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): notas de la versión de 2.6.91.0 y 2.6.92.0, problemas conocidos de la 2.6.84.0, tabla de retirada y plazo del 7 de abril de 2027.

2.  [Microsoft Entra Connect: Upgrade from a previous version](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): métodos de actualización, Swing Migration mediante un servidor de ensayo y aviso de actualización obligatoria.

3.  [Authenticate to Microsoft Entra ID by using application identity](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): cambio manual de servidores existentes a ABA, `Get-ADSyncEntraConnectorCredential` y eliminación de la cuenta heredada.
