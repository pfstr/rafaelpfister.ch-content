---
title: "Microsoft To Do se retira: migración a Planner desde la perspectiva del administrador"
navTitle: "To Do se retira"
description: "Microsoft retirará las aplicaciones To Do el 30 de junio de 2027. Las cuentas profesionales pasarán a Planner, las tareas existentes no se migrarán y Planner almacenará las nuevas tareas privadas fuera del buzón de Exchange. Para los tenants suizos, esto modifica la ubicación de los datos y la cobertura de cumplimiento; los usuarios afectados pueden identificarse mediante Microsoft Graph."
date: "2026-11-24"
kategorie: "Microsoft 365 / Exchange"
timeToRead: "9 min de lectura"
themen:
  - microsoft-365-exchange
produkte:
  - "planner"
  - "exchange-online"
protokolle:
  - "migration"
  - "powershell"
hauptthema: "microsoft-365-exchange"
slug: "microsoft-to-do-se-retira-migracion-a-planner-desde-la-perspectiva-del-administrador"
translationId: "article-7691ba2aae63c942"
translationOf: microsoft-to-do-planner-umstieg
url: https://rafaelpfister.ch/es/blog/microsoft-to-do-se-retira-migracion-a-planner-desde-la-perspectiva-del-administrador
translationSourceHash: ae78db5df03b2677ad2bda294efdd3b1f31ee233d2de5ea0238c2de865396fe8
translationModel: gpt-5.6-terra
translatedAt: 2026-11-24T12:59:37.443Z
translationReview: automatic
---

# Microsoft To Do se retira: migración a Planner desde la perspectiva del administrador

Microsoft retira Microsoft To Do. Las aplicaciones para web, Windows, Mac, iOS y Android se desactivarán el 30 de junio de 2027, tanto para cuentas profesionales y educativas como para cuentas personales de Microsoft. Para las cuentas profesionales y educativas, Microsoft Planner asumirá las tareas personales; la vista revisada «Mis tareas» (My tasks) se implementará a partir de diciembre de 2026. Microsoft publicó el cambio en noviembre de 2026 en un artículo de soporte con calendario, diferencias e indicaciones para administradores.

Conclusión breve: no habrá migración automática. Las tareas existentes de To Do seguirán visibles en Planner hasta el 30 de junio de 2027 y permanecerán en el buzón de Exchange Online. A partir de la implementación, Planner almacenará las nuevas tareas privadas en un plan privado de Planner, es decir, ya no en Exchange. Esto modifica la ubicación de los datos y la cobertura de Purview para los tenants suizos. Revise antes de mediados de 2027 las licencias, las listas compartidas, las integraciones y las obligaciones de conservación. Las cuentas personales de Microsoft no recibirán sustituto.

## Calendario

| Fecha | Cambio |
|---|---|
| Diciembre de 2026 | Inicio de la implementación de la vista revisada «Mis tareas» en Planner para cuentas profesionales y educativas. A partir de entonces, Planner almacenará las tareas privadas recién creadas en un plan privado de Planner. |
| antes del 30 de junio de 2027 | Microsoft proporcionará una función de exportación para los datos de To Do. El formato, el alcance y los requisitos aún no se han publicado. |
| 30 de junio de 2027 | Se retirarán las aplicaciones To Do para web, Windows, Mac, iOS y Android. Después, las tareas existentes de To Do tampoco serán visibles en Planner. |
| más adelante, sin fecha todavía | El servicio de To Do, las API y la sincronización se retirarán por separado. Microsoft comunicará las fechas de forma independiente. |

Planner ya está disponible para cuentas profesionales en la web en `planner.cloud.microsoft` y como aplicación en Teams. Microsoft anuncia que adaptará los puntos de acceso existentes de To Do en Microsoft 365. Microsoft aclaró a Thurrott que las integraciones existentes, entre ellas Outlook y Teams, pasarán a Planner.

## Las tareas existentes no se migrarán

Microsoft no transferirá las tareas ni las listas existentes de To Do a Planner. Hasta el 30 de junio de 2027 aún podrán visualizarse y editarse en Planner; después ya no. No se transferirán al nuevo plan privado.

Los datos no se eliminarán al finalizar las aplicaciones. Según Microsoft, la retirada de las aplicaciones no contempla la eliminación; Microsoft trata por separado la conservación y el acceso. To Do almacena las tareas como tareas de Exchange en el buzón de Exchange Online del usuario, el mismo buzón que contiene correos, citas, contactos y notas. Por ello, el inventario permanecerá inicialmente en el buzón después del 30 de junio de 2027.

El Outlook clásico no se ve afectado por la retirada. Microsoft indica que su función de tareas está separada de To Do, sigue funcionando y se sincroniza con Exchange. Las listas de To Do aparecen en el Outlook clásico como carpetas de tareas en «Mis tareas»; por tanto, los usuarios con Outlook clásico podrán seguir leyendo allí el inventario. Aún no se ha descrito qué ocurrirá con los datos cuando Microsoft retire posteriormente el propio servicio de To Do.

## Las nuevas tareas se almacenan fuera de Exchange

Con la nueva vista cambia la ubicación de almacenamiento. Según las preguntas frecuentes para administradores de Planner, las tareas de To Do y Outlook se encuentran en Exchange, mientras que los planes y sus tareas se almacenan en Azure. Planner almacena los adjuntos de las tareas de los planes en el sitio de SharePoint del grupo, y los planes Premium en Dataverse. Por tanto, las nuevas tareas privadas de «Mis tareas» ya no se encuentran en el buzón del usuario. Esto tiene dos consecuencias para los administradores.

### Ubicación de los datos

Exchange Online figura entre los servicios con un compromiso permanente de ubicación de datos. Si un tenant se crea en Suiza, según los Product Terms Microsoft almacena el contenido de los buzones en reposo (texto de correo, entradas de calendario y adjuntos) únicamente en Suiza. El compromiso no menciona expresamente las tareas, pero estas se encuentran en el mismo buzón.

Microsoft incluye Planner entre los demás servicios de Microsoft 365 sin compromiso permanente. Para tenants con Suiza como país, la tabla indica «EUR1», la macrorregión Europa, como ubicación de almacenamiento. Además, Planner no participa en Advanced Data Residency. Las tareas personales que To Do almacenaba hasta ahora en el buzón en Suiza, Planner las almacenará tras el cambio en la macrorregión Europa. Las ubicaciones de almacenamiento por servicio para su tenant figuran en la tarjeta de ubicación de datos (Data Location Card) del Centro de administración de Microsoft 365. Las organizaciones que hayan garantizado contractualmente o mediante directrices internas una ubicación de datos en Suiza deben evaluar este cambio.

### Conservación y eDiscovery

Las directivas de conservación de Exchange en Microsoft Purview solo incluyen las tareas del buzón si tienen una fecha de finalización; las tareas sin fecha de finalización no están cubiertas. Esta limitación ya se aplica actualmente a las tareas de To Do.

Según Microsoft, para Planner Purview solo admite la búsqueda de eDiscovery y Legal Hold para tareas de planes compartidos con un grupo. No se admiten expresamente los planes personales (planes de lista), los planes en componentes de Loop, los planes Premium ni los planes en canales compartidos y privados de Teams. Microsoft implementa desde mediados de julio de 2026 las directivas de conservación y eliminación para contenido de Planner; se anunció su finalización para principios de agosto (MC1384412). Según el anuncio, pueden aplicarse a planes basados en usuarios y grupos. El anuncio no indica a qué tipo de plan corresponde técnicamente el nuevo plan privado. Tras la implementación, compruebe si sus directivas de conservación y sus retenciones incluyen las nuevas tareas privadas.

## Diferencias entre To Do y Planner

Microsoft enumera las siguientes diferencias:

| Área | Diferencia según Microsoft |
|---|---|
| Organización | Planner reúne las tareas privadas y el trabajo asignado en «Mis tareas», actualmente con vista de cuadrícula y tablero. La vista de lista llegará con la revisión. |
| Recordatorios | Las notificaciones de fecha de vencimiento de Planner difieren de los recordatorios programados por tarea de To Do. |
| Uso compartido | Planner funciona con planes y asignaciones. No se transferirán las membresías, las asignaciones ni los permisos de las listas compartidas de To Do. |
| Escritorio y sin conexión | Planner está disponible en la web y en Teams. No se prevén aplicaciones nativas para Windows y Mac; la web y Teams no ofrecen una función sin conexión comparable. |
| Categorías | Las etiquetas de Planner funcionan de forma distinta a las categorías de Outlook y no se sincronizan con ellas. |
| Adjuntos | Actualmente no se admiten adjuntos de archivos en tareas del nuevo plan privado de Planner. |
| Estrella y prioridad | La estrella personal sirve para la planificación propia y llegará con la nueva vista. La prioridad de una tarea se aplica a todos los participantes de un plan compartido. |
| Grupos de listas | No tienen equivalente directo en Planner. |
| Imprimir, enviar lista por correo | Actualmente no está previsto para la solución de sustitución. |

Los usuarios que trabajan sin conexión con la aplicación To Do en Windows o Mac no tendrán un sustituto equivalente en Planner.

## Licencias y acceso a Planner

Planner está incluido, entre otros, en Microsoft 365 Business Basic, Business Standard y Business Premium, así como en Office 365 E1, E3 y E5. Si Planner no está incluido en una licencia, Microsoft ofrece Planner Plan 1 o Planner and Project Plan 3 como suscripciones independientes; también son necesarias para las funciones Premium. Según Microsoft, no se requiere ninguna configuración nueva y no existe un interruptor específico con el que los administradores puedan controlar o aplazar el cambio.

Si Planner está desactivado en la asignación de licencias (plan de servicio `PROJECTWORKMANAGEMENT`), los usuarios afectados necesitarán volver a tener acceso si Planner debe sustituir a To Do. Según las preguntas frecuentes de Planner, eliminar la licencia de Planner solo oculta el mosaico: mediante `planner.cloud.microsoft`, los usuarios sin licencia pueden seguir creando y editando planes. Si hasta ahora ha bloqueado Planner mediante la licencia, revise esta decisión en vista del cambio.

## Inventario con Microsoft Graph

Es posible consultar mediante la API de To Do de Microsoft Graph qué usuarios utilizan To Do y qué listas están compartidas. El siguiente script escribe en un archivo CSV todas las listas por usuario, con el estado de uso compartido y el número de tareas abiertas. Requiere los módulos `Microsoft.Graph.Authentication` y `Microsoft.Graph.Users`, así como un registro de aplicación con los permisos de aplicación `Tasks.Read.All` y `User.Read.All` (con consentimiento de administrador) y un certificado para iniciar sesión. `Tasks.Read.All` concede acceso de lectura a las tareas de todos los usuarios; elimine de nuevo el permiso o el registro de aplicación después del inventario.

```powershell
$verbindung = @{
    ClientId              = '<App-ID>'
    TenantId              = '<Tenant-ID>'
    CertificateThumbprint = '<Zertifikat-Fingerabdruck>'
}
Connect-MgGraph @verbindung -NoWelcome

$benutzer = Get-MgUser -All -Filter "accountEnabled eq true" -Property Id, UserPrincipalName

$bestand = foreach ($b in $benutzer) {
    try {
        $listen = Get-MgUserTodoList -UserId $b.Id -All -ErrorAction Stop
    }
    catch {
        # Sin buzón de Exchange Online o sin acceso: omitir usuario
        continue
    }
    foreach ($l in $listen) {
        $aufgaben = @(Get-MgUserTodoTask -UserId $b.Id -TodoTaskListId $l.Id -All)
        [pscustomobject]@{
            Benutzer = $b.UserPrincipalName
            Liste    = $l.DisplayName
            Typ      = $l.WellknownListName
            Geteilt  = $l.IsShared
            Besitzer = $l.IsOwner
            Offen    = @($aufgaben | Where-Object Status -ne 'completed').Count
            Gesamt   = $aufgaben.Count
        }
    }
}

$bestand | Export-Csv -Path .\todo-bestand.csv -NoTypeInformation -Encoding UTF8
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `ClientId`, `TenantId` | Registro de aplicación y tenant en los que inicia sesión el script |
| `CertificateThumbprint` | Huella digital del certificado en el almacén de certificados del usuario actual; inicio de sesión como aplicación sin cuenta de usuario |
| `-NoWelcome` | Suprime el mensaje de bienvenida de `Connect-MgGraph` |
| `-All` | Lee todas las páginas de un resultado en lugar de solo la primera |
| `-Filter "accountEnabled eq true"` | Solo cuentas activadas; se excluyen las cuentas desactivadas, como los buzones compartidos |
| `-Property Id, UserPrincipalName` | Carga solo las dos propiedades necesarias |
| `-UserId` | Usuario cuyas listas o tareas se consultan |
| `-ErrorAction Stop` | Hace que un error pueda capturarse para que `catch` pueda omitir al usuario |
| `-TodoTaskListId` | Lista cuyas tareas se consultan |
| `WellknownListName` | `defaultList` para la lista estándar «Tareas», `flaggedEmails` para mensajes marcados y `none` para listas propias |
| `IsShared` | `True` si la lista está compartida con otros usuarios |
| `IsOwner` | `False` si la lista pertenece a otro usuario y solo se ha compartido |
| `Status -ne 'completed'` | Cuenta todas las tareas no completadas |
| `-NoTypeInformation` | Sin línea de tipo al inicio del archivo CSV |
| `-Encoding UTF8` | Se conservan los caracteres especiales en los nombres de las listas |

</details>

Las cuentas sin buzón de Exchange Online devuelven un error y se omiten. En tenants grandes conviene realizar una prueba con un subconjunto, por ejemplo con `-Top 20` en lugar de `-All` en `Get-MgUser`.

El análisis del archivo CSV:

- **Listas compartidas** (`Geteilt` = `True`) muestran dónde se realiza la colaboración mediante To Do. Microsoft no transferirá estas membresías; las listas deben recrearse como un plan de Planner con las personas correspondientes.
- **Tareas abiertas** en listas propias (`Typ` = `none` o `defaultList`) deben ser transferidas manualmente por los usuarios a Planner antes del 30 de junio de 2027 o protegidas mediante la función de exportación anunciada.
- **Mensajes marcados** (`Typ` = `flaggedEmails`) se conservarán como marca en el correo del buzón.

## Cuentas personales de Microsoft

Las aplicaciones To Do también se retirarán el 30 de junio de 2027 para las cuentas personales de Microsoft, incluidas Microsoft 365 Personal y Family. Planner no está disponible para estas cuentas. Microsoft recomienda elegir otro gestor de tareas y proteger los datos con la función de exportación anunciada. Según Microsoft, una exportación no significa que los datos puedan importarse en Planner u otra aplicación. Los usuarios con cuenta profesional y personal deben revisar ambas cuentas por separado.

## Preparación para administradores

1. **Acceso y licencias:** asegúrese de que todos los usuarios de To Do tengan una licencia con Planner y que el plan de servicio `PROJECTWORKMANAGEMENT` esté activado.

2. **Listas compartidas e integraciones:** recopile el inventario con el script anterior. Identifique los flujos de Power Automate y las aplicaciones de terceros que usan la API de To Do de Microsoft Graph. Microsoft retirará la API por separado y más adelante; según Microsoft, las integraciones pueden requerir ajustes de todos modos.

3. **Conservación y ubicación de datos:** compruebe si existen obligaciones de conservación aplicables a las tareas de To Do y si las directivas y retenciones de Purview incluyen el nuevo plan privado. Evalúe si la ubicación de almacenamiento en la macrorregión Europa es compatible con sus requisitos. Antes de eliminar o reorganizar datos de tareas, consulte a los responsables de cumplimiento.

4. **Informar a los usuarios:** comunique a los usuarios que no habrá migración, durante cuánto tiempo seguirá visible el inventario en Planner y qué funciones cambiarán (uso sin conexión, recordatorios, adjuntos, grupos de listas).

5. **Seguir el Centro de mensajes:** Microsoft anunciará allí los detalles de la implementación, la función de exportación y las fechas para el servicio y las API.

## Fuentes

1.  [Moving from Microsoft To Do to Microsoft Planner](https://support.microsoft.com/en-us/planner/to-do-to-planner): anuncio con calendario, diferencias entre To Do y Planner, pasos de preparación y preguntas frecuentes.

2.  [Review data storage and compliance in Microsoft To Do](https://support.microsoft.com/en-us/todo/review-data-storage-and-compliance-in-microsoft-to-do): almacenamiento de las tareas de To Do como tareas en el buzón de Exchange Online.

3.  [Frequently asked questions for admins about Microsoft Planner](https://learn.microsoft.com/en-us/planner/faq-for-planner-admins): ubicaciones de almacenamiento de las tareas (Exchange, Azure, SharePoint, Dataverse), plan de servicio `PROJECTWORKMANAGEMENT` y acceso mediante `planner.cloud.microsoft` sin licencia.

4.  [Microsoft Planner for admins](https://learn.microsoft.com/en-us/planner/planner-for-admins): suscripciones con Planner, así como Planner Plan 1 y Planner and Project Plan 3.

5.  [Other Microsoft 365 services data locations](https://learn.microsoft.com/en-us/microsoft-365/enterprise/m365-dr-service-other?view=o365-worldwide#planner): Planner sin compromiso permanente de ubicación de datos, ubicación «EUR1» para tenants con Suiza como país.

6.  [Data Residency for Exchange Online](https://learn.microsoft.com/en-us/microsoft-365/enterprise/m365-dr-service-exo?view=o365-worldwide): compromiso de los Product Terms para el contenido de buzones en reposo en Suiza.

7.  [Security, privacy, and compliance in Microsoft Planner](https://learn.microsoft.com/en-us/planner/planner-security-privacy-compliance): almacenamiento de los datos de planes en Azure; Planner no participa en Advanced Data Residency. Estado de agosto de 2025, antes de la introducción de las directivas de conservación para Planner.

8.  [Microsoft Planner support in Microsoft Purview](https://learn.microsoft.com/en-us/planner/planner-in-purview): eDiscovery y Legal Hold solo para tareas en planes basados en grupos, lista de tipos de planes no compatibles.

9.  [MC1384412: Retention policies for Microsoft Planner content](https://mc.merill.net/message/MC1384412): copia del aviso del Centro de mensajes sobre directivas de conservación y eliminación para Planner, implementación de julio a agosto de 2026.

10.  [Learn about retention for Exchange](https://learn.microsoft.com/en-us/purview/retention-policies-exchange): elementos de buzón compatibles; tareas solo con fecha de finalización.

11.  [Use the Microsoft To Do API](https://learn.microsoft.com/en-us/graph/api/resources/todo-overview?view=graph-rest-1.0): recursos de la API de To Do en Microsoft Graph (listas, tareas, listas de comprobación, recursos vinculados).

12.  [List lists](https://learn.microsoft.com/en-us/graph/api/todo-list-lists?view=graph-rest-1.0): punto de conexión, permisos y propiedades `isShared`, `isOwner` y `wellknownListName`.

13.  [List Todo tasks](https://learn.microsoft.com/en-us/graph/api/todotasklist-list-tasks?view=graph-rest-1.0): punto de conexión para las tareas de una lista y permiso de aplicación `Tasks.Read.All`.

14.  [Microsoft Will Retire its To Do App in June 2027](https://www.thurrott.com/cloud/344071/microsoft-will-retire-its-to-do-app-on-in-june-2027): informe con una declaración posterior de Microsoft sobre las integraciones con Outlook y Teams.
