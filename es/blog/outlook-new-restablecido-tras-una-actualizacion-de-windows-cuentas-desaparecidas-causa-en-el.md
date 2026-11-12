---
title: "Outlook New restablecido tras una actualización de Windows: cuentas desaparecidas, causa en el registro"
navTitle: "Restablecimiento de Outlook"
description: "El nuevo Outlook para Windows se inicia tras reiniciar para instalar una actualización como si acabara de instalarse; faltan todas las cuentas y configuraciones. La causa puede rastrearse en el registro Olk: una comprobación activada en el servidor (Flight nh-userDataSentinel) detecta la ausencia de un archivo marcador y elimina los datos locales sin preguntar. Este artículo muestra las entradas del registro, cómo demostrar el caso en sus propios dispositivos y qué hacer después del restablecimiento."
date: "2026-11-11"
kategorie: "Outlook"
timeToRead: "6 min de lectura"
themen:
  - outlook
  - windows-client
produkte:
  - "outlook"
  - "windows-client"
protokolle:
  - "troubleshooting"
slug: "outlook-new-restablecido-tras-una-actualizacion-de-windows-cuentas-desaparecidas-causa-en-el"
translationId: "article-c1340c0bc1862647"
translationOf: outlook-new-reset-nach-update
url: https://rafaelpfister.ch/es/blog/outlook-new-restablecido-tras-una-actualizacion-de-windows-cuentas-desaparecidas-causa-en-el
translationSourceHash: 405d93eb98233f046e7ecb2823969a601db0041bb6ddaef8eeaa4c7d858e393c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-12T03:28:07.685Z
translationReview: automatic
---

# Outlook New restablecido tras una actualización de Windows: cuentas desaparecidas, causa en el registro

Tras el reinicio para las actualizaciones de Windows de noviembre de 2026, el nuevo Outlook para Windows se inició en uno de mis dispositivos como después de una instalación nueva. La cuenta profesional, configurada como cuenta principal, había desaparecido, al igual que las demás cuentas, las firmas y la configuración personal. No apareció ningún mensaje de error; Outlook mostró directamente el asistente para agregar una cuenta.

La causa figura en el registro local de Outlook: la aplicación eliminó ella misma sus datos locales porque una comprobación recién activada evaluó el estado como «restablecimiento necesario». No hay indicios de intervención por parte de Windows Update, un usuario o un atacante.

## Secuencia

| Hora (hora local) | Evento | Fuente |
|---|---|---|
| 18:26 | Última entrada de la sesión de Outlook en ejecución, sin cierre ordenado | `Olk\logs\olk_*.log` |
| 18:27 y 18:28 | Dos reinicios por Windows Update (`MoNotificationUx.exe`, después `TrustedInstaller.exe`) | Registro del sistema, evento 1074 |
| 18:41:55 | Outlook se inicia, la comprobación `nh-userDataSentinel` está activa e informa de `ResetRequired` | nuevo `olk_*.log` |
| 18:41:56 | Outlook elimina sus datos locales y se inicia como una instalación nueva | nuevo `olk_*.log` |

Las marcas de tiempo del registro Olk y de los nombres de archivo de los registros están en UTC. En la tabla se han convertido a hora local (CET).

## Las entradas del registro

Las primeras líneas de la nueva sesión muestran la comprobación y el resultado:

```text
Checking flight enabled status  flightName nh-userDataSentinel  isEnabled true
Reset sentinel check completed.  installState ResetRequired
  settingsFound true  resetSentinelFlagFound true
Reset sentinel check triggered user data reset.
Prompted for clear local state  clearLocalStateName Silent
```

A continuación se producen las operaciones de eliminación, cada una con `succeeded true`:

| Ruta bajo `%LOCALAPPDATA%\Microsoft\Olk` | Contenido |
|---|---|
| `EBWebView` | Perfil de WebView2 con estado de inicio de sesión, lista de cuentas y datos sin conexión |
| `UserSettings.json` | Configuración de la aplicación |
| `Attachments` | Archivos adjuntos almacenados localmente en caché |
| `pst_index_v2` | Índice de archivos PST vinculados |
| `cache\Notification`, `cache\*.db` | Caché de notificaciones y de procesos en segundo plano |
| `updated.txt`, `xpdApi.log`, `Feedback` | Archivos de estado y auxiliares |

Unas líneas más tarde aparece `Last update date file missing, creating it. Barring clearing local state, this is likely a fresh install`. A partir de ese momento, Outlook se comporta como una instalación nueva.

## Por qué se activó la comprobación

`nh-userDataSentinel` es un Flight, es decir, una función que Microsoft activa del lado del servidor para instalaciones individuales. En todos los registros anteriores del dispositivo, este Flight figura como `Flight status not set; using default (false)`, por última vez a las 13:23 UTC del mismo día. Al iniciarse a las 17:41 UTC, estaba activo por primera vez.

La comprobación busca un archivo marcador en la carpeta del paquete de la aplicación:

```text
%LOCALAPPDATA%\Packages\Microsoft.OutlookForWindows_8wekyb3d8bbwe\LocalState\OlkSentinel
```

Si Outlook encuentra configuración de usuario en la carpeta Olk (`settingsFound true`), pero no un marcador correspondiente en la carpeta del paquete, lo interpreta como indicio de que la aplicación se ha restablecido o reinstalado y elimina los datos de usuario. En el dispositivo afectado, `OlkSentinel` se creó de nuevo a las 18:41:56; los demás archivos de `LocalState` datan de septiembre de 2026. Por tanto, la carpeta del paquete no se restableció; el archivo marcador simplemente no existía antes porque la comprobación era nueva. En el primer inicio con el Flight activo, el resultado fue por ello un restablecimiento, aunque no era necesario.

Según los registros, el reinicio por Windows Update es solo el momento, no la causa: el Flight se evalúa al iniciar, y el primer inicio tras la actualización fue el primero con el Flight activo. La versión de la aplicación (1.2026.1026.100) era la misma antes y después del reinicio.

## Cómo demostrar el caso en sus propios dispositivos

La siguiente consulta busca en todos los registros Olk las entradas de la comprobación:

```powershell
$logs = "$env:LOCALAPPDATA\Microsoft\Olk\logs"
Get-ChildItem $logs -Filter 'olk_*.log' |
  Select-String -Pattern 'Reset sentinel|Delete directory tree' |
  Select-Object Filename, LineNumber, Line
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `-Filter 'olk_*.log'` | solo los registros de sesión de la aplicación, sin archivos ZIP ni registros push |
| `-Pattern` | expresión regular; coincide con la comprobación y con cada operación de eliminación |
| `Select-Object Filename, LineNumber, Line` | muestra para cada coincidencia el archivo de registro, el número de línea y el texto |

</details>

Outlook empaqueta las sesiones anteriores en archivos ZIP en la misma carpeta. Si el restablecimiento se produjo hace varios días, es posible que la entrada solo esté en el archivo y deba descomprimirse antes.

Como segundo indicio sirven las horas de creación: si `EBWebView` y `UserSettings.json` en la carpeta Olk, así como `OlkSentinel` en la carpeta del paquete, tienen la misma marca de tiempo, el restablecimiento se produjo en ese momento.

```powershell
$olk = "$env:LOCALAPPDATA\Microsoft\Olk"
$pkg = "$env:LOCALAPPDATA\Packages\Microsoft.OutlookForWindows_8wekyb3d8bbwe"
Get-Item "$olk\EBWebView", "$olk\UserSettings.json", "$pkg\LocalState\OlkSentinel" |
  Select-Object FullName, CreationTime
```

## Qué se pierde y qué no

Solo se eliminaron datos locales de la aplicación. Los buzones, calendarios y contactos se encuentran en Microsoft 365 y Outlook.com en el servidor, y vuelven a aparecer al iniciar sesión de nuevo. El inicio de sesión en Windows (Web Account Manager) se mantiene; Outlook ofrece directamente al agregar una cuenta las cuentas de Microsoft almacenadas en Windows.

Debe volver a configurar:

- todas las cuentas, en el orden deseado (la primera cuenta agregada es la cuenta principal; véase más abajo)
- las firmas almacenadas localmente, si no vuelven mediante la sincronización en la nube de la cuenta
- los archivos PST vinculados
- la configuración de la aplicación, como la vista, las notificaciones y el comportamiento de inicio

En el caso de la cuenta principal, el orden es decisivo, ya que el nuevo Outlook ofrece algunas funciones solo en la cuenta principal, como S/MIME (véase [Firma S/MIME en la cuenta secundaria](/blog/outlook-smime-sekundaeres-konto-anhaenge-fehlen)). Por ello, agregue primero la cuenta que deba ser la principal. Si Outlook ya ha adoptado automáticamente otra cuenta después del restablecimiento, elimínela, agregue la cuenta principal deseada y vuelva a vincular las demás cuentas después.

## Para administradores

El restablecimiento no se puede deshacer después de que ocurra; realizar una copia de seguridad de la carpeta `%LOCALAPPDATA%\Microsoft\Olk` antes del inicio es la única forma de recuperar el estado anterior. En entornos con contenedores de perfil (FSLogix) y sesiones VDI no persistentes, debe comprobar si tanto la carpeta Olk como la carpeta del paquete `Microsoft.OutlookForWindows_8wekyb3d8bbwe` se conservan en el perfil. Si solo se conserva una de las dos carpetas, corresponde exactamente al estado que la comprobación interpreta como una instalación nueva.

Los informes de usuarios de que el nuevo Outlook «ha olvidado todas las cuentas» pueden clasificarse en pocos minutos con la consulta del registro anterior. Si aparece `Reset sentinel check triggered user data reset`, no se trata de un problema de cuenta ni de un incidente de seguridad.

## Fuentes

1.  [Microsoft Learn: Deploy the new Outlook for Windows on VDI](https://learn.microsoft.com/en-us/microsoft-365-apps/outlook/get-started/virtualized-desktop-infrastructure): ubicaciones de los datos locales del nuevo Outlook e indicaciones sobre contenedores de perfil.

2.  [Microsoft Learn: Control installing and using the new Outlook](https://learn.microsoft.com/en-us/microsoft-365-apps/outlook/get-started/control-install): distribución y control de la aplicación en entornos administrados.

3.  [Microsoft Q&A: Outlook (new) won't open, makes me attempt to sign in](https://learn.microsoft.com/en-us/answers/questions/5863503/outlook-new-wont-open-makes-me-attempt-to-sign-in): informes de usuarios sobre cuentas perdidas tras restablecer los datos locales.
