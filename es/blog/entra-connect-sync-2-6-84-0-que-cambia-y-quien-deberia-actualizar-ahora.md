---
slug: "entra-connect-sync-2-6-84-0-que-cambia-y-quien-deberia-actualizar-ahora"
title: "Entra Connect Sync 2.6.84.0: qué cambia y quién debería actualizar ahora"
navTitle: "Entra Connect 2.6.84"
description: "La versión de seguridad incorpora compatibilidad con passkeys y cambios en la autenticación de aplicaciones, PowerShell y Password Hash Sync. La versión anterior fue retirada; por ello, la actualización requiere una decisión escalonada."
date: "2026-07-17"
kategorie: "Microsoft Entra"
timeToRead: "11 min de lectura"
themen:
  - microsoft-entra
  - active-directory-entra
draft: false
translationOf: "entra-connect-2-6-84-0"
url: "https://rafaelpfister.ch/es/blog/entra-connect-sync-2-6-84-0-que-cambia-y-quien-deberia-actualizar-ahora"
---

# Entra Connect Sync 2.6.84.0: qué cambia y quién debería actualizar ahora

Microsoft publicó Entra Connect Sync 2.6.84.0 el 7 de julio de 2026 como versión de seguridad y recomienda actualizar cuanto antes. Al mismo tiempo, se retiró el predecesor directo 2.6.79.0 debido a un problema del instalador descubierto posteriormente. La consecuencia no es ni «instalar inmediatamente en todas partes» ni «esperar e ignorar»: los sistemas afectados y los que pronto quedarán fuera de soporte deberían cambiar con rapidez, mientras que el resto puede probar primero la actualización de forma controlada.

## Por qué esta versión merece especial cautela

La línea 2.6 de Entra Connect Sync ha tenido un inicio accidentado. Un breve repaso, porque es relevante para decidir sobre la actualización:

- **2.6.1.0** (febrero de 2026) corrigió, entre otros, un error por el que editar la configuración del conector de Entra ID en Synchronization Service Manager eliminaba los parámetros de Application-Based Authentication, lo que provocaba fallos en el asistente y en la rotación de certificados. Por ello, para todas las versiones 2.5 se aplicaba la llamativa recomendación de simplemente no utilizar la interfaz de administración del producto.
- **2.6.3.0** (marzo de 2026) fue un hotfix para un problema por el que la actualización automática podía detener inesperadamente el servidor de Entra Connect. La solución provisional de entonces: la actualización automática detecta archivos de configuración modificados manualmente y simplemente omite esos servidores.
- **2.6.79.0** (junio de 2026) fue retirada por completo después de su publicación. El instalador ya no está disponible; según Microsoft, quien tenga instalada esta versión debe desinstalarla e instalar 2.6.84.0. Microsoft no documenta cuál fue exactamente el problema.

A fecha de hoy, la versión 2.6.84.0 solo está disponible para descargar desde el Centro de administración de Microsoft Entra («Released for download»). Todavía no se ha anunciado un despliegue mediante actualización automática. Esto también es una señal: Microsoft aún no distribuye la versión de forma generalizada a instalaciones existentes.

## Nuevas funciones

### Inicio de sesión resistente al phishing en el asistente de instalación (vista previa)

El asistente de instalación ahora admite el inicio de sesión con passkeys y claves de seguridad FIDO2 mediante Windows Web Account Manager (WAM). El contexto es el siguiente: desde 2024/2025, Microsoft está exigiendo gradualmente MFA para los inicios de sesión en las interfaces de administración de Azure y Entra, y muchas organizaciones han restringido sus cuentas administrativas mediante Acceso condicional a métodos resistentes al phishing (FIDO2, passkeys, autenticación basada en certificados). Precisamente estas cuentas correctamente protegidas no podían iniciar sesión hasta ahora en el asistente de Entra Connect, porque el cuadro de diálogo de inicio de sesión integrado no admitía estos métodos. En la práctica, esto llevó a soluciones poco elegantes: por ejemplo, cuentas específicas de «configuración» con requisitos de autenticación más débiles, únicamente para que el asistente pudiera completarse. Esta brecha se cierra ahora, aunque de momento como vista previa.

### Compatibilidad con la nube soberana francesa

La versión 2.6.84.0 incorpora compatibilidad con el entorno de nube soberana francesa, incluida la autenticación de paso, el inicio de sesión único fluido, la escritura diferida de contraseñas y la supervisión con el agente de Health. En línea con ello, se corrigió un error por el que el nombre de nube de Application Proxy no se resolvía correctamente en France Cloud y el registro de PTA fallaba con «EnvironmentName attribute is invalid».

## Cambios de comportamiento en detalle

La parte más interesante de esta versión no son las nuevas funciones, sino los comportamientos modificados. Varios de ellos corrigen decisiones de diseño que han causado sorpresas en la práctica.

### La actualización automática ya no destruye archivos de configuración personalizados

Este es el cambio con la historia más larga. Hasta ahora, la actualización automática sobrescribía por completo el archivo `miiserver.exe.config` durante la actualización. Los ajustes manuales se perdían. Parecía un caso marginal, pero no lo era: Microsoft había indicado a los administradores de entornos FIPS que editaran precisamente este archivo para que Password Hash Synchronization funcionara con el modo FIPS activado. Por tanto, quien siguió la guía oficial tenía un archivo de configuración «modificado».

Las consecuencias se manifestaron al actualizar a 2.5.190.0 y 2.6.1.0 como un problema conocido: si el instalador detecta un archivo `miiserver.exe.config` modificado, deja el archivo intacto; pero entonces falta el nuevo enlace de ensamblado y el servicio de sincronización deja de funcionar tras la actualización con `System.IO.FileLoadException: Could not load file or assembly 'System.Diagnostics.DiagnosticSource, Version=6.0.0.1'`. La solución alternativa documentada: añadir manualmente un bindingRedirect en la sección `assemblyBinding` de `miiserver.exe.config` (en `%programfiles%\Microsoft Azure AD Sync\Bin`):

```xml
<dependentAssembly>
  <assemblyIdentity name="System.Diagnostics.DiagnosticSource" publicKeyToken="cc7b13ffcd2ddd51" culture="neutral" />
  <bindingRedirect oldVersion="0.0.0.0-8.0.0.0" newVersion="8.0.0.0" />
</dependentAssembly>
```

Después, reinicie el servicio ADSync. El hotfix 2.6.3.0 solo mitigó el problema para la actualización automática: los servidores afectados simplemente se omitían y permanecían en la versión anterior. Con 2.6.84.0 llega la solución real: el proceso de actualización fusiona las personalizaciones del cliente con la nueva configuración y valida el resultado antes de aplicarlo. No obstante, quien actualice manualmente desde una versión afectada debería comprobar antes el estado de su `miiserver.exe.config` y hacer una copia de seguridad del archivo: el mecanismo de fusión es nuevo y, por tanto, todavía no está probado en la práctica.

### Application-Based Authentication: fin de la reversión silenciosa y del cambio silencioso

Como recordatorio: desde 2.5.76.0, Application-Based Authentication (ABA) tiene disponibilidad general y es el estándar. En lugar de la antigua cuenta de Directory Synchronization (una cuenta en la nube con contraseña almacenada), el servidor de sincronización se autentica como aplicación de Entra ID con un certificado, idealmente protegido por TPM. Esta es una arquitectura mucho más robusta: no hay una contraseña que pueda filtrarse y la credencial está vinculada a la máquina.

La versión 2.6.84.0 corrige dos comportamientos que socavaban esta mejora de seguridad:

**Ya no hay reversión silenciosa.** Si la configuración de ABA fallaba en el asistente, hasta ahora la instalación volvía sin avisar a la cuenta heredada. El resultado: el administrador creía disponer de un inicio de sesión basado en certificados, pero en realidad el servidor funcionaba con la antigua cuenta con contraseña. Un patrón clásico de fail-open. Ahora el asistente se detiene con un mensaje de error claro («Microsoft Entra Connect could not configure application-based authentication for this server. Setup cannot continue.»), para que se corrija la causa real en lugar de ocultarla.

**Ya no hay conversión automática en segundo plano.** Hasta ahora, Entra Connect convertía por sí solo los servidores existentes de la cuenta heredada a ABA durante la operación de sincronización en curso. Bienintencionado desde el punto de vista de la seguridad, pero una pesadilla desde el punto de vista operativo: un método de autenticación cambia sin consultar, sin ventana de cambios y sin que nadie lo sepa. Y si algo falla durante el proceso (problemas de TPM, conflictos de Acceso condicional, firewall), la sincronización se detiene. Ahora se aplica lo siguiente: solo las instalaciones nuevas configuran ABA automáticamente; los servidores existentes no cambian hasta que un administrador inicia el asistente y selecciona explícitamente **Configure application-based authentication to Microsoft Entra ID**. El cambio vuelve así al lugar al que pertenece: un cambio planificado.

Además, se ha mejorado el tratamiento del TPM: la instalación ahora prueba previamente la capacidad de firma de un certificado y gestiona correctamente la comprobación de firmas del TPM. En servidores con firmware TPM defectuoso que no puede generar una firma válida, la instalación recurre de forma controlada a un certificado basado en software. Esto también tiene antecedentes: los fallos de ABA relacionados con TPM se prolongaron durante varias versiones anteriores (2.5.79.0, 2.5.190.0), entre otras causas por incompatibilidades entre implementaciones de TPM y el procedimiento de firma estándar de la biblioteca MSAL.

### Los cmdlets de PowerShell ahora exigen un inicio de sesión explícito de administrador

Un cambio que los responsables de scripts deben tener presente: los cmdlets `Set-ADSyncAADCompanyFeature` y `Set-ADSyncAADPasswordSyncState`, que modifican la configuración en la nube, ahora requieren el parámetro `-AADUsername` para una autenticación interactiva de administrador. El propio asistente tampoco escribe ya cambios en la nube con credenciales de servicio almacenadas, sino mediante un inicio de sesión interactivo de MSAL. Asimismo, el asistente de desinstalación solicita credenciales de administrador para limpiar la configuración en la nube; si se omite este paso, solo se realiza la limpieza local.

El trasfondo es el mismo hilo conductor que con ABA: las acciones contra el tenant deben poder asignarse a una identidad de administrador real y trazable, en lugar de a una cuenta de servicio anónima. Esto encaja con una corrección de errores de la misma versión: hasta ahora, el registro de auditoría de administrador anotaba la identidad de la cuenta de servicio en lugar del administrador que realmente actuaba cuando se modificaban reglas de sincronización; un rastro de auditoría que no cumplía su propósito. Solo ambas medidas juntas proporcionan una auditoría útil. La consecuencia práctica: quien hasta ahora ejecutaba estos cmdlets sin supervisión en scripts debe rediseñar esos procesos: la autenticación interactiva y la automatización no son compatibles.

### Se elimina la autorreparación de PHS

El cambio más discreto, pero conceptualmente interesante: Password Hash Synchronization ya no reactiva por sí misma en segundo plano su marca de característica en la nube. Si la marca está desactivada, un administrador debe volver a activarla explícitamente.

Hasta ahora, si PHS se desactivaba a nivel de tenant —de forma consciente o accidental—, la función se «reparaba» sola y volvía a activarse. Para entornos que habían desactivado PHS deliberadamente —por ejemplo, por motivos de cumplimiento, porque no pueden fluir hashes de contraseñas a la nube, o durante una fase de migración—, era una función que anulaba una decisión documentada de administrador. Era difícil de justificar que precisamente un mecanismo que sincroniza hashes de contraseñas se reactivara por iniciativa propia.

Sin embargo, no se debe ocultar la otra cara: la autorreparación también salvó entornos en los que la marca se desactivó por un error o un script fallido, sin que nadie lo advirtiera. Esta protección desaparece ahora. Quien use PHS en producción —aunque solo sea como alternativa para el inicio de sesión de emergencia— debería supervisar activamente el estado de PHS en el futuro, por ejemplo mediante Entra Connect Health o revisando los valores de heartbeat de la sincronización.

### Componentes actualizados: SQL LocalDB 2022, MSAL y runtime de VC++

Menos espectacular, pero necesario desde hace tiempo, es la modernización de los componentes incluidos:

- **SQL Server LocalDB 2019 → 2022.** La base de datos interna de Entra Connect se basaba hasta ahora en SQL Server 2019 Express LocalDB, una versión cuyo soporte estándar finalizó en febrero de 2025. Con SQL Server 2022, la instalación vuelve a estar en una versión con soporte vigente.
- **MSAL 4.64.1 → 4.83.3.** Microsoft Authentication Library es el componente central para toda obtención de tokens (ABA, inicio de sesión en el asistente, PowerShell). El salto de unas veinte versiones menores incorpora las correcciones y mejoras acumuladas de la biblioteca.
- **Visual C++ Redistributable 2013 → 2015–2022 (14.42).** Aquí resulta menos destacable la actualización que la carga heredada: hasta esta versión, Entra Connect requería un entorno de ejecución cuyo soporte terminó en abril de 2024. La dependencia de VC++ 2013 se ha eliminado por completo.

Esto coincide con la indicación general de las notas de la versión de que se han corregido «multiple security vulnerabilities in bundled third-party dependencies». Probablemente sea la razón principal para clasificarla como versión de seguridad: los componentes incluidos obsoletos no son un problema cosmético en un producto que funciona con permisos cercanos a los de administrador de dominio en el centro de la infraestructura de identidad.

## Las demás correcciones de errores

Para completar, estas son las correcciones restantes:

- **Búsqueda de metaverso en Synchronization Service Manager** reparada. Tras la advertencia de no utilizar la interfaz en versiones anteriores, ahora parece que vuelve a recibir mantenimiento.
- **Informe de diagnóstico de PowerShell (HTML)** vuelve a renderizarse correctamente; relevante para todos los que usan `Invoke-ADSyncDiagnostics` en casos de soporte.
- **Conector Generic SQL:** la creación de perfiles fallaba porque no se rellenaban parámetros obligatorios durante la configuración. Afecta a entornos que conectan directorios adicionales mediante el conector GSQL.
- **China Cloud:** el nombre de instancia no se resolvía correctamente mediante la API del punto de conexión de detección, lo que podía hacer fallar la detección de instancias de nube.
- **Registro de auditoría de administrador** ahora registra al administrador real en lugar de la cuenta de servicio cuando se modifican reglas de sincronización (véase arriba).

## Plazos de soporte: quién debe actuar de todos modos ahora

Desde marzo de 2023 se aplica una política de retirada estricta para Entra Connect Sync 2.x: cada versión queda fuera de soporte doce meses después de la aparición de la versión sucesora. Los plazos actuales:

| Versión | Fin del soporte |
| --- | --- |
| 2.5.3.0 | **31 de julio de 2026** |
| 2.5.76.0 | 1 de septiembre de 2026 |
| 2.5.79.0 | 23 de octubre de 2026 |
| 2.5.190.0 | 2 de febrero de 2027 |
| 2.6.1.0 | 10 de marzo de 2027 |
| 2.6.3.0 | 7 de julio de 2027 |

Quien todavía use 2.5.3.0 dispone, por tanto, de solo dos semanas de soporte. Aquí la pregunta no es si actualizar, sino a qué versión. Microsoft también recalca que las versiones fuera de soporte pueden dejar de funcionar «unexpectedly»; en el caso de las versiones 1.x retiradas, la sincronización ya se ha desactivado efectivamente en el lado del servidor. Los requisitos mínimos siguen siendo .NET Framework 4.7.2 y TLS 1.2; el instalador está disponible exclusivamente en el Centro de administración de Entra (Entra ID → Entra Connect → Get started), ya no en el Centro de descargas.

## Recomendación según la versión de partida

Microsoft recomienda actualizar «lo antes posible». Sin embargo, esta recomendación figuraba literalmente también sobre la versión 2.6.79.0, la que posteriormente fue retirada. El historial reciente de versiones —instalador retirado, hotfix debido a servidores detenidos, advertencias de interfaz durante varias versiones— justifica una evaluación sobria en lugar de una reacción automática.

Mi valoración para entornos típicos:

**Esperar algunas semanas es defendible** si utiliza una versión aún compatible (2.5.190.0 o posterior), ninguno de los problemas corregidos le afecta de forma urgente y no necesita ninguna de las nuevas funciones. Según las notas de la versión, las vulnerabilidades de seguridad corregidas se encuentran en componentes de terceros incluidos; un servidor de Entra Connect debería estar en cualquier caso tan aislado —sin acceso a Internet salvo a los puntos de conexión de Microsoft, sin inicios de sesión interactivos, con tratamiento Tier 0— que este margen de tiempo pueda justificarse. Si la versión permanece algunas semanas sin ser retirada y Microsoft inicia el despliegue de actualización automática, será una señal de calidad mucho mejor que cualquier anuncio.

**Debería actuar con rapidez** si se cumple alguno de estos puntos:

- **Tiene instalada la versión 2.6.79.0.** En ese caso, la instrucción es inequívoca: desinstálela e instale 2.6.84.0; no espere.
- **Utiliza 2.5.3.0** (fin de soporte el 31 de julio de 2026) o una versión aún más antigua cuyo soporte ya haya expirado.
- **Alguno de los problemas corregidos le afecta específicamente**, por ejemplo, la configuración de ABA en servidores TPM, el conector GSQL o el requisito de auditoría de que los cambios de reglas se asignen al administrador correcto.

Para la propia actualización se aplica el procedimiento habitual, especialmente recomendable con este historial de versiones: exporte antes la configuración (el asistente ofrece **View or export current configuration**), instale primero la actualización en un servidor en modo de ensayo y compruebe allí los ciclos de sincronización, el asistente y la rotación de certificados; solo después, actualice el servidor activo. Quien tenga un `miiserver.exe.config` personalizado debe hacer una copia de seguridad antes de actualizar y comprobar después si el nuevo mecanismo de fusión ha adoptado correctamente las personalizaciones. Y quien ejecute scripts con `Set-ADSyncAADCompanyFeature` o `Set-ADSyncAADPasswordSyncState` debe probarlos antes del despliegue en producción; de lo contrario, fallarán debido al nuevo parámetro obligatorio.

## Fuentes

1. [Microsoft Entra Connect: Version release history – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): Notas oficiales de la versión 2.6.84.0, incluida la indicación de retirada de 2.6.79.0, la tabla de retirada y el problema conocido con miiserver.exe.config modificado.
1. [Microsoft Entra Connect: Upgrade from a previous version to the latest – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): Procedimiento de actualización, incluida la migración swing mediante un servidor en modo de ensayo.
1. [Authenticate to Microsoft Entra ID by using application identity – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): Funcionamiento de Application-Based Authentication, que sustituye a la cuenta de servicio heredada.
1. [Microsoft Entra Connect: Phishing-resistant authentication – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-passwordless-authentication): El nuevo inicio de sesión con passkey/FIDO2 en el asistente de instalación mediante Windows Web Account Manager.
1. [Microsoft Entra Connect: Automatic upgrade – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-automatic-upgrade): Mecanismo y requisitos de la actualización automática, cuyo despliegue para 2.6.84.0 todavía está pendiente.
1. [Auditing administrator events in Microsoft Entra Connect Sync – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/admin-audit-logging): El registro de auditoría de administrador, cuya asignación de identidad en reglas de sincronización se corrigió en esta versión.
1. [SQL Server 2019 – Microsoft Lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/sql-server-2019): Fechas de soporte de la base de LocalDB incluida hasta ahora, cuyo soporte estándar finalizó en febrero de 2025.
