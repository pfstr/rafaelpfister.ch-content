---
slug: "entra-connect-sync-2-6-84-0-que-cambia-y-quien-deberia-actualizar-ahora"
title: "Entra Connect Sync 2.6.84.0: qué cambia y quién debería actualizar ahora"
navTitle: "Entra Connect 2.6.84"
description: "La versión de seguridad incorpora compatibilidad con passkeys y cambios en la autenticación de aplicaciones, PowerShell y Password Hash Sync. La versión anterior se ha retirado; por ello, la actualización requiere una decisión gradual."
date: "2026-07-17"
kategorie: "Microsoft Entra"
timeToRead: "11 min de lectura"
themen:
  - microsoft-entra
  - active-directory-entra
draft: false
translationOf: "entra-connect-2-6-84-0"
translationId: article-85bd27acb917e406
translationReview: required
translationSourceHash: da16eeec10c227af5ba6f33ae138e0148db5b34736874eeed6b2b60c0b469a81
translatedAt: 2026-09-05T07:45:42.740Z
url: https://rafaelpfister.ch/es/blog/entra-connect-sync-2-6-84-0-que-cambia-y-quien-deberia-actualizar-ahora
translationModel: gpt-5.6-terra
---

# Entra Connect Sync 2.6.84.0: qué cambia y quién debería actualizar ahora

Microsoft publicó Entra Connect Sync 2.6.84.0 el 7 de julio de 2026 como versión de seguridad y recomienda actualizar cuanto antes. Al mismo tiempo, la predecesora directa, 2.6.79.0, fue retirada debido a un problema del instalador detectado posteriormente. La consecuencia no es ni «instalarla inmediatamente en todas partes» ni «esperar e ignorarla»: los sistemas afectados y los que pronto quedarán sin soporte deberían migrar con rapidez; los demás pueden probar primero la actualización de forma controlada.

## Por qué esta versión merece especial cautela

La rama 2.6 de Entra Connect Sync ha tenido un comienzo accidentado. Un breve repaso, ya que es relevante para decidir sobre la actualización:

- **2.6.1.0** (febrero de 2026) corrigió, entre otros, un error por el que editar la configuración del conector de Entra ID en Synchronization Service Manager eliminaba los parámetros de Application-Based Authentication, provocando errores en el asistente y en la rotación de certificados. Por ello, para todas las versiones 2.5 se aplicaba la llamativa recomendación de simplemente no utilizar la interfaz de administración del producto.
- **2.6.3.0** (marzo de 2026) fue una corrección urgente para un problema por el que Auto-Upgrade podía detener inesperadamente el servidor de Entra Connect. La solución temporal entonces: Auto-Upgrade detecta archivos de configuración modificados manualmente y simplemente omite esos servidores.
- **2.6.79.0** (junio de 2026) fue retirada por completo tras su publicación. El instalador ya no está disponible; según Microsoft, quien tenga instalada esta versión debe desinstalarla e instalar la 2.6.84.0. Microsoft no documenta cuál era exactamente el problema.

A fecha de hoy, la versión 2.6.84.0 solo está disponible para descargar desde el Centro de administración de Microsoft Entra («Released for download»). Todavía no se ha anunciado un despliegue de Auto-Upgrade. También es una señal: Microsoft aún no está distribuyendo la versión de forma generalizada en instalaciones existentes.

## Nuevas funciones

### Inicio de sesión resistente al phishing en el asistente de instalación (versión preliminar)

El asistente de instalación ahora admite el inicio de sesión con passkeys y claves de seguridad FIDO2 mediante Windows Web Account Manager (WAM). El contexto: desde 2024/2025, Microsoft impone gradualmente MFA para los inicios de sesión en interfaces de administración de Azure y Entra, y muchas organizaciones han restringido sus cuentas de administrador mediante Conditional Access a métodos resistentes al phishing (FIDO2, passkeys, autenticación basada en certificados). Hasta ahora, precisamente estas cuentas correctamente protegidas no podían iniciar sesión en el asistente de Entra Connect porque el diálogo de inicio de sesión integrado no admitía esos métodos. En la práctica, esto llevaba a soluciones poco satisfactorias: por ejemplo, cuentas de «configuración» específicas con requisitos de autenticación más débiles, solo para que el asistente pudiera completarse. Esta brecha se cierra ahora, aunque de momento como versión preliminar.

### Compatibilidad con la Sovereign Cloud francesa

La versión 2.6.84.0 incorpora compatibilidad con el entorno Sovereign Cloud francés, incluidas Pass-through Authentication, Seamless Single Sign-On, Password Writeback y la supervisión mediante Health Agent. Asimismo, se ha corregido un error por el que el nombre de nube de Application Proxy no se resolvía correctamente en France Cloud y el registro de PTA fallaba con «EnvironmentName attribute is invalid».

## Cambios de comportamiento en detalle

La parte más interesante de esta versión no son las nuevas funciones, sino los cambios de comportamiento. Varios de ellos corrigen decisiones de diseño que en la práctica habían causado sorpresas.

### Auto-Upgrade ya no destruye archivos de configuración personalizados

Este es el cambio con la historia más larga. Hasta ahora, Auto-Upgrade sobrescribía completamente el archivo `miiserver.exe.config` durante la actualización. Se perdían los ajustes manuales. Parece un caso marginal, pero no lo era: Microsoft había indicado a los administradores en entornos FIPS que editaran precisamente este archivo para que Password Hash Synchronization funcionara con el modo FIPS activado. Por tanto, quien siguió las instrucciones oficiales tenía un archivo de configuración «modificado».

Las consecuencias aparecieron al actualizar a 2.5.190.0 y 2.6.1.0 como un problema conocido: si el instalador detecta un archivo `miiserver.exe.config` modificado, lo deja intacto; pero entonces falta el nuevo enlace de ensamblado y el servicio de sincronización falla tras la actualización con `System.IO.FileLoadException: Could not load file or assembly 'System.Diagnostics.DiagnosticSource, Version=6.0.0.1'`. La solución alternativa documentada: añadir manualmente un bindingRedirect en la sección `assemblyBinding` de `miiserver.exe.config` (en `%programfiles%\Microsoft Azure AD Sync\Bin`):

```xml
<dependentAssembly>
  <assemblyIdentity name="System.Diagnostics.DiagnosticSource" publicKeyToken="cc7b13ffcd2ddd51" culture="neutral" />
  <bindingRedirect oldVersion="0.0.0.0-8.0.0.0" newVersion="8.0.0.0" />
</dependentAssembly>
```

Después, reinicie el servicio ADSync. La corrección urgente 2.6.3.0 solo mitigó el problema para Auto-Upgrade: los servidores afectados simplemente se omitían y permanecían en la versión anterior. Con la 2.6.84.0 llega la solución real: el proceso de actualización combina las personalizaciones del cliente con la nueva configuración y valida el resultado antes de aplicarlo. Quien realice una actualización manual desde una versión afectada debería comprobar previamente el estado de su `miiserver.exe.config` y hacer una copia de seguridad del archivo: el mecanismo de combinación es nuevo y, por tanto, todavía no está probado en la práctica.

### Application-Based Authentication: fin de la reversión silenciosa y del cambio silencioso

Como recordatorio: desde la versión 2.5.76.0, Application-Based Authentication (ABA) es de disponibilidad general y el estándar. En lugar de la antigua cuenta de Directory Synchronization Accounts (una cuenta en la nube con contraseña almacenada), el servidor de sincronización se autentica como aplicación de Entra ID mediante un certificado, idealmente protegido por TPM. Es una arquitectura mucho más sólida: no hay una contraseña que pueda filtrarse y la credencial está vinculada a la máquina.

La versión 2.6.84.0 corrige dos comportamientos que habían socavado esta mejora de seguridad:

**Ya no hay reversión silenciosa.** Si la configuración de ABA fallaba en el asistente, hasta ahora la instalación volvía sin avisar a la cuenta heredada. El resultado: el administrador creía contar con un inicio de sesión basado en certificados, pero el servidor funcionaba realmente con la antigua cuenta con contraseña. Un patrón clásico de fail-open. Ahora, el asistente se detiene con un mensaje de error claro («Microsoft Entra Connect could not configure application-based authentication for this server. Setup cannot continue.»), para que se corrija la causa real en vez de ocultarla.

**Ya no hay cambio automático en segundo plano.** Hasta ahora, Entra Connect migraba por sí solo los servidores existentes de la cuenta heredada a ABA durante la sincronización en curso. Bienintencionado desde el punto de vista de la seguridad, pero un riesgo considerable desde el operativo: el método de autenticación cambia sin consulta, sin ventana de cambio y sin que nadie lo sepa. Y si algo falla durante el proceso (problemas de TPM, conflictos de Conditional Access o firewall), se detiene la sincronización. Ahora se aplica lo siguiente: solo las instalaciones nuevas configuran ABA automáticamente; los servidores existentes no cambian hasta que un administrador inicia el asistente y selecciona explícitamente **Configure application-based authentication to Microsoft Entra ID**. Así, el cambio vuelve a estar donde corresponde: en un cambio planificado.

Además, se ha mejorado el tratamiento de TPM: ahora la instalación prueba de antemano la capacidad de firma de un certificado y gestiona correctamente la comprobación de firma TPM. En servidores con firmware TPM defectuoso que no puede generar una firma válida, la instalación recurre de forma controlada a un certificado basado en software. Esto también tiene antecedentes: los fallos de ABA relacionados con TPM se prolongaron durante varias versiones anteriores (2.5.79.0, 2.5.190.0), entre otras razones por incompatibilidades entre implementaciones de TPM y el procedimiento de firma predeterminado de la biblioteca MSAL.

### Los cmdlets de PowerShell ahora requieren un inicio de sesión explícito de administrador

Un cambio que deben conocer quienes operan scripts: los cmdlets `Set-ADSyncAADCompanyFeature` y `Set-ADSyncAADPasswordSyncState`, que modifican la configuración en la nube, ahora requieren el parámetro `-AADUsername` para una autenticación interactiva de administrador. El propio asistente tampoco realiza ya cambios en la nube con credenciales de servicio almacenadas, sino mediante un inicio de sesión interactivo de MSAL. Además, el asistente de desinstalación solicita credenciales de administrador para limpiar la configuración en la nube; si se omite este paso, la limpieza se realiza solo localmente.

El trasfondo es el mismo hilo conductor que con ABA: las acciones contra el tenant deben poder atribuirse a una identidad de administrador real y trazable, en lugar de a una cuenta de servicio anónima. Esto encaja con una corrección de errores de la misma versión: hasta ahora, el registro de auditoría de administrador anotaba la identidad de la cuenta de servicio en lugar de la del administrador que realizaba realmente los cambios en las reglas de sincronización; un rastro de auditoría que no cumple su propósito. Solo ambas cosas juntas proporcionan una auditoría útil. La consecuencia práctica: quien hasta ahora haya llamado a estos cmdlets sin supervisión en scripts debe rediseñar estos procesos; la autenticación interactiva y la automatización no son compatibles.

### Se elimina la autorreparación de PHS

El cambio más discreto, pero conceptualmente más interesante: Password Hash Synchronization ya no reactiva por sí misma su marca de característica en la nube en segundo plano. Si la marca está desactivada, un administrador debe volver a activarla explícitamente.

Hasta ahora ocurría lo siguiente: si PHS se desactivaba a nivel de tenant, de forma deliberada o accidental, la función se «autorreparaba» y volvía a activarse. Para entornos que habían desactivado PHS intencionadamente —por ejemplo, por motivos de cumplimiento, porque no se permite que los hashes de contraseñas lleguen a la nube, o durante una fase de migración— era una función que anulaba una decisión documentada del administrador. Era difícil de justificar que precisamente un mecanismo que sincroniza hashes de contraseñas se reactivara por su cuenta.

Sin embargo, no debe ocultarse la contrapartida: la autorreparación también salvó entornos en los que la marca se desactivó por un error o por un script fallido sin que nadie lo advirtiera. Esta protección desaparece ahora. Quien utilice PHS en producción —aunque solo sea como respaldo para el inicio de sesión de emergencia— debería supervisar activamente el estado de PHS en el futuro, por ejemplo mediante Entra Connect Health o revisando los valores de heartbeat de la sincronización.

### Componentes actualizados: SQL LocalDB 2022, MSAL y runtime de VC++

Menos espectacular, pero necesaria, es la modernización de los componentes incluidos:

- **SQL Server LocalDB 2019 → 2022.** La base de datos interna de Entra Connect se basaba hasta ahora en SQL Server 2019 Express LocalDB, una versión cuyo soporte estándar terminó en febrero de 2025. Con SQL Server 2022, la instalación vuelve a estar en una versión con soporte vigente.
- **MSAL 4.64.1 → 4.83.3.** Microsoft Authentication Library es el componente central para toda obtención de tokens (ABA, inicio de sesión del asistente, PowerShell). El salto de unas veinte versiones menores incorpora las correcciones y mejoras acumuladas de la biblioteca.
- **Visual C++ Redistributable 2013 → 2015–2022 (14.42).** Aquí es menos destacable la actualización que la carga heredada: hasta esta versión, Entra Connect requería un entorno de ejecución cuyo soporte finalizó en abril de 2024. La dependencia de VC++ 2013 ya se ha eliminado por completo.

A ello se suma la indicación general de las notas de la versión de que se han corregido «multiple security vulnerabilities in bundled third-party dependencies». Probablemente sea el motivo principal de la clasificación como versión de seguridad: los componentes incluidos obsoletos no son un problema cosmético en un producto que opera con permisos cercanos a Domain Admin en el centro de la infraestructura de identidad.

## Las demás correcciones de errores

Por completitud, las restantes correcciones:

- **Búsqueda en el metaverso en Synchronization Service Manager** reparada. Tras la advertencia de no usar la interfaz en versiones anteriores, ahora parece que vuelve a mantenerse.
- **Informe de diagnóstico de PowerShell (HTML)** vuelve a renderizarse correctamente; relevante para todos los que utilizan `Invoke-ADSyncDiagnostics` en casos de soporte.
- **Conector Generic SQL:** la creación de perfiles fallaba porque no se rellenaban parámetros obligatorios durante la configuración. Afecta a entornos que conectan directorios adicionales mediante el conector GSQL.
- **China Cloud:** el nombre de instancia no se resolvía correctamente mediante la API de endpoint de detección, lo que podía hacer fallar la detección de la instancia de nube.
- **Registro de auditoría de administrador** ahora registra el administrador real, en lugar de la cuenta de servicio, al modificar reglas de sincronización (véase arriba).

## Plazos de soporte: quién debe actuar de todos modos ahora

Desde marzo de 2023 se aplica una estricta política de retirada para Entra Connect Sync 2.x: cada versión queda sin soporte doce meses después de la aparición de su sucesora. Los plazos actuales:

| Versión | Fin del soporte |
| --- | --- |
| 2.5.3.0 | **31 de julio de 2026** |
| 2.5.76.0 | 1 de septiembre de 2026 |
| 2.5.79.0 | 23 de octubre de 2026 |
| 2.5.190.0 | 2 de febrero de 2027 |
| 2.6.1.0 | 10 de marzo de 2027 |
| 2.6.3.0 | 7 de julio de 2027 |

Por tanto, quien todavía utilice la 2.5.3.0 dispone de solo dos semanas de soporte. La cuestión aquí no es si actualizar, sino a qué versión. Microsoft también subraya que las versiones sin soporte pueden dejar de funcionar «unexpectedly»; en las versiones 1.x retiradas, la sincronización ya se ha desactivado realmente del lado del servidor. Los requisitos mínimos siguen siendo .NET Framework 4.7.2 y TLS 1.2; el instalador está disponible exclusivamente en el Centro de administración de Entra (Entra ID → Entra Connect → Get started), ya no en Download Center.

## Recomendación según la versión de partida

Microsoft recomienda actualizar «lo antes posible». Sin embargo, esta recomendación figuraba exactamente igual para la versión 2.6.79.0, la versión que posteriormente fue retirada. El historial reciente de versiones —instalador retirado, corrección urgente por servidores detenidos y advertencias sobre la interfaz en varias versiones— justifica una evaluación sobria en lugar de un reflejo.

Mi valoración para entornos típicos:

**Esperar algunas semanas es aceptable** si ejecuta una versión todavía compatible (2.5.190.0 o posterior), ninguno de los problemas corregidos le afecta de forma urgente y no necesita ninguna de las nuevas funciones. Según las notas de la versión, las vulnerabilidades corregidas se encuentran en componentes de terceros incluidos; un servidor de Entra Connect debería estar de todos modos tan aislado —sin acceso a Internet excepto a los endpoints de Microsoft, sin inicios de sesión interactivos y con tratamiento Tier 0— que este margen de tiempo pueda justificarse. Si la versión permanece algunas semanas sin ser retirada y Microsoft inicia el despliegue de Auto-Upgrade, será una señal de calidad mucho mejor que cualquier anuncio.

**Debe actuar con rapidez** si se cumple alguno de estos puntos:

- **Tiene instalada la 2.6.79.0.** En ese caso, la instrucción es inequívoca: desinstale e instale la 2.6.84.0; no espere.
- **Utiliza la 2.5.3.0** (fin de soporte: 31 de julio de 2026) o una versión aún más antigua que ya ha expirado.
- **Uno de los problemas corregidos le afecta concretamente**, como la configuración de ABA en servidores TPM, el conector GSQL o el requisito de auditoría de que los cambios de reglas se asignen al administrador correcto.

Para la actualización propiamente dicha se aplica el procedimiento habitual, especialmente recomendable dada la historia de estas versiones: exporte previamente la configuración (el asistente ofrece **View or export current configuration**), aplique primero la actualización en un servidor en modo de almacenamiento provisional y compruebe allí los ciclos de sincronización, el asistente y la rotación de certificados; solo después actualice el servidor activo. Quien tenga un `miiserver.exe.config` personalizado debe hacer una copia de seguridad antes de actualizar y comprobar después si el nuevo mecanismo de combinación ha incorporado correctamente las personalizaciones. Quien ejecute scripts con `Set-ADSyncAADCompanyFeature` o `Set-ADSyncAADPasswordSyncState` debe probarlos antes del despliegue en producción; de lo contrario, fallarán por el nuevo parámetro obligatorio.

## Fuentes

1. [Microsoft Entra Connect: Version release history – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/reference-connect-version-history): Notas oficiales de la versión 2.6.84.0, incluida la indicación de retirada de la 2.6.79.0, la tabla de retirada y el problema conocido con miiserver.exe.config modificado.
1. [Microsoft Entra Connect: Upgrade from a previous version to the latest – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-upgrade-previous-version): Procedimiento de actualización, incluida la migración swing mediante un servidor en modo de almacenamiento provisional.
1. [Authenticate to Microsoft Entra ID by using application identity – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/authenticate-application-id): Funcionamiento de Application-Based Authentication, que sustituye a la cuenta de servicio heredada.
1. [Microsoft Entra Connect: Phishing-resistant authentication – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-passwordless-authentication): El nuevo inicio de sesión con passkey/FIDO2 en el asistente de instalación mediante Windows Web Account Manager.
1. [Microsoft Entra Connect: Automatic upgrade – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-automatic-upgrade): Mecanismo y requisitos de Auto-Upgrade, cuyo despliegue para 2.6.84.0 sigue pendiente.
1. [Auditing administrator events in Microsoft Entra Connect Sync – Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/admin-audit-logging): El registro de auditoría de administrador, cuya asignación de identidad para las reglas de sincronización se ha corregido en esta versión.
1. [SQL Server 2019 – Microsoft Lifecycle](https://learn.microsoft.com/en-us/lifecycle/products/sql-server-2019): Fechas de soporte de la base LocalDB incluida hasta ahora, cuyo soporte estándar terminó en febrero de 2025.
