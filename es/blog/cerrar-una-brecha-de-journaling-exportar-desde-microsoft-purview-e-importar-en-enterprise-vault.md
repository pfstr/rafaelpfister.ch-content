---
title: "Cerrar una brecha de journaling: exportar desde Microsoft Purview e importar en Enterprise Vault"
navTitle: "Brecha de journaling"
description: "Si los informes de diario de Exchange Online fallan durante algunos días, no pueden generarse posteriormente. Sin embargo, el contenido sigue estando en los buzones. Cómo exportarlo como PST mediante la nueva eDiscovery de Purview, por qué el migrador PST de Enterprise Vault generalmente no es una opción y cómo realizar la reimportación a través del buzón de diario sin desplazar el flujo de diario en curso, con dos scripts genéricos para importar, mover y registrar."
date: "2026-09-22"
kategorie: "Archivado y journaling"
timeToRead: "18 min de lectura"
themen:
  - archivierung-journaling
  - microsoft-365-exchange
  - exchange-onprem-hybrid
produkte:
  - "exchange-online"
  - "exchange-on-premises"
protokolle:
  - "backup-dr"
  - "powershell"
  - "troubleshooting"
slug: "cerrar-una-brecha-de-journaling-exportar-desde-microsoft-purview-e-importar-en-enterprise-vault"
translationId: "article-5b24118ac7567d96"
aiPrompt: |
  Du bist mein Exchange- und Archivierungsassistent. Für einen Zeitraum sind keine Journalberichte im Enterprise-Vault-Journalarchiv angekommen. Hilf mir Schritt für Schritt: Zeitraum der Lücke aus den Postfachstatistiken bestimmen, Aufbewahrungsrichtlinien prüfen, die Suche in der neuen Purview-eDiscovery mit korrekt geklammerter KQL-Abfrage anlegen, den Export als PST je Postfach ohne Ordnerstruktur konfigurieren, die PST-Dateien per New-MailboxImportRequest in das Journalpostfach importieren, die importierten Unterordner per EWS in den Posteingang leeren und den Nachimport so drosseln, dass der laufende Journalstrom Vorrang behält. Weise mich auf die Grenzen hin: fehlende Umschlagempfänger und Mehrfachkopien ohne Entdopplung.
translationOf: journaling-luecke-purview-export-enterprise-vault-import
url: https://rafaelpfister.ch/es/blog/cerrar-una-brecha-de-journaling-exportar-desde-microsoft-purview-e-importar-en-enterprise-vault
translationSourceHash: 532fe3454abb44c445145c6fd3e7424685e8bc5a28dfa18d25cff8a7f5516094
translationModel: gpt-5.6-terra
translatedAt: 2026-09-24T08:40:46.476Z
translationReview: automatic
---

# Cerrar una brecha de journaling: exportar desde Microsoft Purview e importar en Enterprise Vault

El journaling es un proceso de transporte. Exchange genera el informe de diario en el momento en que el mensaje pasa por el transporte y lo entrega al buzón de diario. Si esta entrega falla, por ejemplo porque un conector o un objeto de destinatario está mal configurado, no existe ninguna función que recupere el informe más tarde. Lo que pasó por el transporte durante ese periodo falta en el archivo.

Sin embargo, el contenido de los mensajes sigue estando en los buzones de los implicados y, con una directiva de retención sin vencimiento, también cuando los usuarios han eliminado los mensajes. De ello se deriva la única vía viable para completar los datos: exportar el periodo desde todos los buzones e importar las copias en el archivo de diario. El siguiente procedimiento utiliza la nueva eDiscovery de Microsoft Purview para la exportación y Veritas Enterprise Vault 15 para la importación; las métricas proceden de una reimportación de varios cientos de miles de mensajes, durante la cual se crearon los dos scripts.

## Lo que una exportación puede sustituir y lo que no

Un informe de diario consta del sobre y del contenido. El sobre contiene los destinatarios a los que el transporte entregó realmente el mensaje: también las copias ocultas y los miembros resueltos de listas de distribución. Una copia de buzón no contiene estos datos. Tras la reimportación ya no se puede determinar quién recibió un mensaje como copia oculta. En cambio, el contenido en sí puede reconstruirse por completo si se cumplen dos condiciones.

En primer lugar, debe existir una retención sobre los buzones para que incluso los mensajes eliminados sigan estando en los elementos recuperables. Puede comprobarlo en PowerShell de seguridad y cumplimiento:

```powershell
Connect-IPPSSession
Get-RetentionCompliancePolicy -DistributionDetail |
  Format-List Name, Enabled, Mode, ExchangeLocation, ExchangeLocationException
Get-RetentionComplianceRule |
  Format-List Name, Policy, RetentionComplianceAction, RetentionDuration
```

Una directiva con `RetentionComplianceAction: Keep` y `RetentionDuration: Unlimited` mediante `ExchangeLocation: All` protege el contenido íntegramente. La lista de excepciones en `ExchangeLocationException` identifica los buzones para los que esto no se aplica. Para ellos, solo puede reconstruirse lo que aún esté disponible.

En segundo lugar, una exportación cuenta copias de buzón, no mensajes. Un mensaje dirigido a quince destinatarios internos aparece quince veces en el resultado. En el caso descrito, para un único día había 98'131 copias frente a unos 6'800 mensajes únicos, determinados mediante el seguimiento de mensajes. Cómo tratar estas copias lo decide Compliance, no TI (véase la sección sobre deduplicación).

## Determinar el periodo

El inicio de la brecha está en el propio buzón de diario, en el servidor Exchange que lo aloja. El momento del último informe entregado marca el inicio:

```powershell
Get-MailboxFolderStatistics "journal@example.com" -IncludeOldestAndNewestItems |
  Where-Object { $_.ItemsInFolder -gt 0 } |
  Format-Table Name, ItemsInFolder, OldestItemReceivedDate, NewestItemReceivedDate -AutoSize
```

Lo mismo se aplica al buzón que figura en `JournalingReportNdrTo` de la configuración de transporte. Recopila informes de diario no entregables y su entrada más reciente confirma el momento. El fin de la brecha es el momento en que la regla de diario se cambió al destino reparado. Entre ambos se encuentra el periodo de exportación, en UTC.

El seguimiento de mensajes proporciona la cifra objetivo para la conciliación: una línea por destinatario y mensaje, con `MessageId`. Un informe histórico cubre todo el periodo:

```powershell
Connect-ExchangeOnline
$bericht = @{
  ReportTitle   = "Journal-Luecke"
  ReportType    = "MessageTrace"
  StartDate     = "2026-09-10"
  EndDate       = "2026-09-12"
  NotifyAddress = "admin@example.com"
}
Start-HistoricalSearch @bericht
```

Los valores únicos de `MessageId` de este informe son la cifra contra la que se mide la reimportación al final.

## Exportación desde la nueva eDiscovery

Microsoft retiró las interfaces clásicas de Content Search y eDiscovery (Standard) el 31 de agosto de 2025. Lo que aún aparece en muchas guías, especialmente la opción de deduplicación durante la exportación, ya no existe en la nueva eDiscovery. El procedimiento en el portal Purview:

1. En *eDiscovery*, cree o abra un caso. Para la exportación, la cuenta ejecutora necesita el rol *eDiscovery Manager* y una licencia Microsoft 365 E3 o E5.
2. Cree una búsqueda. Fuente de datos: todos los buzones o, para una prueba, un único buzón.
3. Use KQL como consulta. Los paréntesis son decisivos, porque `AND` tiene mayor precedencia que `OR`:

```text
kind:email AND ((received>=2026-09-10 AND received<2026-09-12)
             OR (sent>=2026-09-10 AND sent<2026-09-12))
```

Sin `kind:email` la búsqueda también cuenta entradas de calendario, contactos y tareas. En el caso descrito, esto supuso la diferencia entre 473'725 y 98'131 resultados para un día. Sin los paréntesis exteriores, la segunda condición de fecha solo se aplica a `sent`, y el resultado contiene mensajes de todo el inventario del buzón.

4. Ejecute *Generate statistics*. En búsquedas de todo el tenant, esto acelera considerablemente la exportación, ya que solo se procesan los buzones con resultados. Antes de exportar, vuelva a procesar las ubicaciones que terminen con error mediante *Retry failed locations*; de lo contrario, faltarán en el resultado y solo se detectará después de la descarga.
5. Seleccione *Export* con los siguientes ajustes.

| Opción | Efecto |
|---|---|
| *Export type: Export items with items report* | Mensajes más `items.csv`; el informe por sí solo no contiene contenido |
| *Export format: Create PSTs for messages* | Un PST por buzón en lugar de archivos `.msg` individuales |
| *Maximum PST package size* | A partir de este tamaño, un PST se divide en partes (`.001.pst`, `.002.pst`) |
| *Maximum .zip package size* | Tamaño de los paquetes de descarga; debe ser al menos igual al tamaño del PST |
| *Organize data from different locations into separate folders or PSTs* | **Activar**: un archivo por buzón, denominado `<smtp-adresse>.001.pst` |
| *Include folder and path of the source* | **Desactivar**: todos los mensajes terminan en la carpeta `Items` del PST, en lugar de en la estructura de carpetas del buzón |
| *Give each item a friendly name* | Sin efecto en la exportación PST |

El ajuste *Include folder and path of the source* determina la importación posterior. Con estructura de carpetas, la importación crea en el buzón de diario las carpetas de cada buzón, en el idioma de cada usuario: `Posteingang`, `Posta in arrivo`, `Boîte de réception`. Sin estructura de carpetas, todo queda en una carpeta `Items`, y la reimportación tiene un único lugar desde el que continuar.

La descarga proporciona paquetes Zip. Microsoft recomienda 7-Zip o una herramienta comparable en lugar del Explorador de Windows. Los paquetes caducan 14 días después de su creación. El archivo `items.csv` del informe de proceso de la exportación (en *Process manager*, Export, informes) documenta qué contiene la exportación y qué se omitió; forma parte del inventario.

### Deduplicación

La búsqueda de contenido clásica incluía la casilla *Enable de-duplication*: los mensajes con iguales `InternetMessageId`, `ConversationTopic` y `BodyTagInfo` se exportaban una sola vez; las demás ubicaciones encontradas figuraban en `Results.csv`. La nueva eDiscovery no ofrece esta opción en la exportación directa desde una búsqueda. La única vía documentada es un *Review Set*: cargar allí los resultados de búsqueda, ejecutar Analytics, utilizar el filtro generado automáticamente *For Review*, que excluye duplicados, y exportar desde el Review Set. Esto requiere eDiscovery Premium y, por tanto, una licencia E5 para el usuario ejecutor. Varios usuarios informan en Microsoft Q&A de duplicados pese a este procedimiento; se recomienda hacer una prueba de un día antes de la exportación completa.

Sin deduplicación se aplica lo siguiente: Enterprise Vault deduplica en el nivel de almacenamiento (los adjuntos y cuerpos de mensaje idénticos se almacenan una sola vez), pero no en el nivel de elemento. Cada copia de buzón se convierte en una entrada de archivo propia, un resultado de búsqueda propio y un contador propio. Por tanto, el recuento para la demostración no puede derivarse del archivo, sino únicamente de `items.csv` frente al seguimiento de mensajes. Esta decisión, aceptar las copias o volver a exportar mediante un Review Set, se toma antes de la importación, no después.

## Las vías hacia Enterprise Vault

Enterprise Vault incluye el migrador PST para archivos PST, como asistente en la consola de administración (*Archives*, clic derecho, *Import PST*) y como variante scriptable mediante el Policy Manager EVPM. Para la reimportación en un archivo de diario, esta vía tiene tres limitaciones.

La primera es la licencia. El migrador requiere la función `EVPSTM` (*Exchange PST Migrator*). En un sitio que solo opera journaling, con frecuencia no está licenciada, y el asistente se interrumpe en la primera página con *Required license not installed*. El Policy Manager utiliza el mismo migrador y muestra el mismo mensaje.

La segunda es el destino. El asistente no ofrece archivos de diario como destino, solo archivos de buzón y archivos de correo de Internet. Una importación directa en el archivo de diario solo es posible mediante EVPM con `ArchiveName`, y con ello se eluden las reglas de procesamiento de Journaling Task. Como destino queda un Shared Archive propio en el Vault Store del diario, cuya búsqueda cubra conjuntamente el archivo de diario.

La tercera es la trazabilidad: el migrador escribe en el archivo PST (marca los elementos migrados), por lo que el inventario se modifica, y los informes de migración de cada archivo deben conciliarse con las cifras de exportación.

La segunda vía no requiere una licencia adicional: Exchange Journaling Task de Enterprise Vault no solo archiva informes de diario. Si identifica un elemento como informe de diario, lo desempaqueta y adopta los datos del sobre. Archiva directamente un elemento normal, igual que siempre hizo en tiempos del journaling simple de Exchange sin sobre. Quien importe los archivos PST en el buzón de diario consigue que el Task regular escriba los mensajes en el archivo de diario regular, con la misma categoría de retención, la misma indexación y sin cambios en la arquitectura de archivado.

Puede demostrarse con un único mensaje de prueba enviado al buzón de diario: si poco después ya no está en ninguna carpeta, ni siquiera en *Invalid Journal Report*, y la búsqueda lo encuentra en el archivo de diario, el Task archiva mensajes normales.

Esta vía tiene dos características que determinan el procedimiento. El Task procesa exclusivamente el nivel superior de la bandeja de entrada, no las subcarpetas. Esto puede verse en el buzón de diario: la carpeta de búsqueda *Initial Trawl With Pending* bajo *Enterprise Vault Search Folders* cuenta exactamente los elementos de la bandeja de entrada; las subcarpetas no se incluyen. Y la importación mediante Exchange siempre crea carpetas; cómo, se explica en la sección siguiente.

## Importación en el buzón de diario

`New-MailboxImportRequest` importa un archivo PST del lado del servidor a través de Mailbox Replication Service. El archivo debe estar en un recurso compartido sobre el que *Exchange Trusted Subsystem* tenga control total, y la cuenta ejecutora necesita el rol *Mailbox Import Export*, que de forma predeterminada no está asignado a nadie:

```powershell
Get-ManagementRoleAssignment -Role "Mailbox Import Export" |
  Format-Table RoleAssigneeName, RoleAssigneeType -AutoSize
New-ManagementRoleAssignment -Role "Mailbox Import Export" -User "admin@example.com"
```

La importación propiamente dicha:

```powershell
$import = @{
  Mailbox          = "journal@example.com"
  FilePath         = "\\EVSERVER01\pstimport$\unzipped\user@example.com.001.pst"
  TargetRootFolder = "Inbox"
  Name             = "NI-user_example.com.001"
  BatchName        = "Nachimport"
}
New-MailboxImportRequest @import
```

<details class="options-details">
<summary>Explicación de las opciones</summary>

| Opción | Efecto |
|---|---|
| `-Mailbox` | Buzón de destino; aquí, el buzón de diario |
| `-FilePath` | Ruta UNC del archivo PST; las rutas locales se rechazan |
| `-TargetRootFolder` | Carpeta bajo la que se deposita el contenido PST; si no se indica, las carpetas PST se asignan a carpetas de buzón con el mismo nombre |
| `-SourceRootFolder` | Carpeta del PST desde la que se importa; en la práctica, la propia carpeta se crea de todos modos (véase el texto) |
| `-Name` | Nombre único del trabajo; para miles de archivos se deriva del nombre de archivo |
| `-BatchName` | Agrupación para consultas y estadísticas |

</details>

Tres observaciones de la prueba determinan el resto del procedimiento. Sin `TargetRootFolder` y con un PST estructurado (estructura de carpetas exportada), MRS crea la raíz del PST como una carpeta propia junto a la bandeja de entrada; en el caso de un buzón alemán, `Oberste-Ebene-des-Informationsspeichers` con las subcarpetas `Posteingang`, `Gesendete-Elemente` y otras, además de `Recoverable-Items` con `Deletions`. Con `TargetRootFolder "Inbox"` y un PST plano, todos los mensajes llegan a `/Inbox/Items`. Y `SourceRootFolder "Items"` en combinación con `TargetRootFolder "Inbox"` también produjo en la prueba `/Inbox/Items`; el nivel no se eliminó.

En los tres casos, los mensajes quedan fuera de la vista de Journaling Task. El último paso, moverlos de forma plana a la bandeja de entrada, no puede realizarse con herramientas nativas de Exchange; se hace mediante EWS. La EWS Managed API se encuentra como `Microsoft.Exchange.WebServices.dll` en el directorio de instalación de Enterprise Vault, y Vault Service Account tiene control total sobre el buzón de diario, por lo que no se necesita permiso adicional. El script se ejecuta en el servidor EV con Windows PowerShell 5.1, porque la biblioteca se basa en .NET Framework.

Este paso de movimiento es a la vez el regulador de todo el procedimiento: determina cuántos mensajes ve el Task de una vez.

## La prueba con un buzón

Antes de procesar miles de archivos, puede comprobarse toda la ruta con un único buzón. Es conveniente usar el buzón de la persona que ejecutó la exportación: tiene la autorización, son sus propios datos y el archivo es pequeño. El procedimiento:

1. Copia del archivo PST en una carpeta independiente, hash SHA-256 del original en un archivo CSV.
2. Importación con `TargetRootFolder "Inbox"`, estadísticas con `Get-MailboxImportRequestStatistics`: `ItemsTransferred` es la cifra objetivo.
3. Estadísticas de carpetas del buzón de diario: ¿dónde están los elementos?
4. Mover desde `/Inbox/Items` a la bandeja de entrada mediante EWS.
5. Recuento de los elementos antiguos en la bandeja de entrada cada pocos minutos: los elementos con `DateTimeReceived` anteriores al inicio del flujo en curso son los importados. Si el número disminuye, el Task está archivando.
6. Conciliación: búsqueda en el archivo de diario (en Discovery Accelerator o en la búsqueda de EV) con periodo y propietario del buzón; comparar el número de resultados con `ItemsTransferred`.

En el caso descrito se importaron 128 elementos y se archivaron 126. Los dos restantes eran un borrador y un elemento sin remitente; el Task deja los elementos sin remitente. Los borradores nunca pasaron por el transporte y no pertenecen a ningún diario, por lo que el script de movimiento los excluye mediante la marca `IsDraft`, independientemente del idioma. Tampoco se mueven elementos de las carpetas *Versions* y *Konflikte* (versiones anteriores bajo retención, restos de sincronización).

El registro de eventos `Veritas Enterprise Vault` muestra después de la prueba lo que el Task rechazó. Los eventos 3071 (*could not be archived as it may be corrupt*) y 3288 (*no longer archive pending*) se repiten en cada pasada para los mismos elementos; estos elementos deberían moverse a una carpeta fuera de la bandeja de entrada para que el Task no vuelva a intentarlo en cada pasada.

## Script 1: Importación por lotes

El primer script se ejecuta en Exchange Management Shell de un servidor Exchange. Crea trabajos de importación por lotes, espera, registra por archivo el estado y `ItemsTransferred` en un archivo CSV y elimina los trabajos completados. Las ejecuciones repetidas omiten archivos que ya figuran en el registro. El prefijo `NI` significa reimportación.

```powershell
$NIPostfach  = "journal@example.com"
$NIQuellen   = @("\\EVSERVER01\pstimport$\unzipped", "\\EVSERVER01\pstimport$\unzipped2")
$NIProtokoll = "\\EVSERVER01\pstimport$\protokoll\import-protokoll.csv"

function Schreibe-NIImport($Ordner, $Datei, $Request, $Status, $Items, $Meldung) {
  [pscustomobject]@{
    Zeit = Get-Date -Format "yyyy-MM-dd HH:mm:ss"; Ordner = $Ordner; Datei = $Datei
    Request = $Request; Status = $Status; Items = $Items; Meldung = $Meldung
  } | Export-Csv -Path $NIProtokoll -Append -NoTypeInformation -Encoding UTF8
}

function Get-NIDateien {
  foreach ($q in $NIQuellen) {
    $o = Split-Path $q -Leaf
    Get-ChildItem $q -Filter *.pst | Sort-Object Name | ForEach-Object {
      [pscustomobject]@{
        Ordner = $o; Datei = $_.Name; Pfad = $_.FullName
        MB = [math]::Round($_.Length / 1MB, 1)
        Request = "NI-" + $o + "-" + ($_.Name -replace "@", "_" -replace "\.pst$", "")
      }
    }
  }
}

function Get-NIErledigt {
  if (Test-Path $NIProtokoll) {
    Import-Csv $NIProtokoll |
      Where-Object { $_.Status -in @("angelegt", "Completed", "CompletedWithWarning") } |
      Select-Object -ExpandProperty Request -Unique
  }
}

function Start-NICharge([int]$Anzahl = 100) {
  $erledigt = @(Get-NIErledigt)
  $alle  = @(Get-NIDateien)
  $offen = @($alle | Where-Object { $erledigt -notcontains $_.Request } | Select-Object -First $Anzahl)
  foreach ($d in $offen) {
    $auftrag = @{
      Mailbox          = $NIPostfach
      FilePath         = $d.Pfad
      TargetRootFolder = "Inbox"
      Name             = $d.Request
      BatchName        = "Nachimport-" + $d.Ordner
      ErrorAction      = "Stop"
    }
    try {
      $null = New-MailboxImportRequest @auftrag
      Schreibe-NIImport $d.Ordner $d.Datei $d.Request "angelegt" "" ""
    } catch {
      Schreibe-NIImport $d.Ordner $d.Datei $d.Request "Fehler-Anlage" "" $_.Exception.Message
    }
  }
  "angelegt: $($offen.Count)   verbleibend: $($alle.Count - $erledigt.Count - $offen.Count)"
}

function Wait-NICharge {
  do {
    $alle = @(Get-MailboxImportRequest -Mailbox $NIPostfach | Where-Object { $_.Name -like "NI-*" })
    $laufend = @($alle | Where-Object { $_.Status -notin @("Completed", "Failed", "CompletedWithWarning") })
    "$(Get-Date -Format HH:mm:ss)  laufend: $($laufend.Count)   fertig: $($alle.Count - $laufend.Count)"
    if ($laufend.Count -gt 0) { Start-Sleep -Seconds 60 }
  } while ($laufend.Count -gt 0)
  foreach ($r in $alle) {
    $s = $r | Get-MailboxImportRequestStatistics
    if (-not $s) { Schreibe-NIImport "" "" $r.Name "Statistik-Fehler" "" ""; continue }
    $ordner = $r.BatchName -replace "^Nachimport-", ""
    $datei  = if ($s.FilePath) { Split-Path $s.FilePath -Leaf } else { "" }
    Schreibe-NIImport $ordner $datei $r.Name $s.Status $s.ItemsTransferred $s.Message
    if ($s.Status -in @("Completed", "CompletedWithWarning")) {
      $r | Remove-MailboxImportRequest -Confirm:$false
    }
  }
  Get-MailboxStatistics $NIPostfach | Format-List ItemCount, TotalItemSize
}

function Resume-NICharge([int]$Anzahl = 200, [int]$MaxLager = 20000) {
  $stat  = Get-MailboxFolderStatistics $NIPostfach
  $lager = ($stat | Where-Object { $_.FolderPath -like "/Inbox/*" } | Measure-Object ItemsInFolder -Sum).Sum
  "in Unterordnern wartend: $lager"
  if ($lager -gt $MaxLager) { "Lager voll, nichts freigegeben"; return }
  Get-MailboxImportRequest -Mailbox $NIPostfach -Status Suspended |
    Select-Object -First $Anzahl |
    Resume-MailboxImportRequest -Confirm:$false
  "freigegeben: $Anzahl"
}

function Get-NIImportStand {
  $p = Import-Csv $NIProtokoll
  $p | Group-Object Status | Select-Object Name, Count | Format-Table -AutoSize
  "Elemente importiert: " + (($p | Where-Object { $_.Status -like "Completed*" } | Measure-Object Items -Sum).Sum)
  "Dateien gesamt: " + @(Get-NIDateien).Count
}
```

<details class="options-details">
<summary>Explicación de las funciones</summary>

| Función | Efecto |
|---|---|
| `Get-NIDateien` | Lista de todos los archivos PST de las carpetas de origen con nombre de trabajo derivado; los mismos nombres de archivo en distintas carpetas (por ejemplo, una exportación por día) siguen siendo distinguibles |
| `Start-NICharge -Anzahl` | Crea hasta N trabajos de importación para archivos que aún no se han registrado |
| `Wait-NICharge` | Espera hasta que no quede ningún trabajo en ejecución, escribe el estado y `ItemsTransferred` en el registro y elimina los trabajos completados; los fallidos permanecen para análisis. Las estadísticas y la eliminación se ejecutan mediante la canalización, porque `-Identity` no acepta el objeto Identity deserializado en una shell remota |
| `Resume-NICharge -Anzahl -MaxLager` | Libera trabajos suspendidos solo si en las subcarpetas de la bandeja de entrada esperan menos de `MaxLager` mensajes |
| `Get-NIImportStand` | Resumen por estado, suma de elementos importados, número total de archivos |

</details>

Debe conocer dos comportamientos de MRS. Exchange permite de forma predeterminada diez trabajos simultáneos para el mismo buzón de destino; todos los demás permanecen en cola con un motivo como `StalledDueToTarget_MailboxCapacityExceeded` o `StalledDueToTarget_MdbReplication` y avanzan después. Se trata de la gestión de carga de trabajo, no de un problema de capacidad, aunque el nombre lo sugiera. Además, `Get-MailboxImportRequest` muestra el estado con retraso; las estadísticas están más actualizadas.

El rendimiento fue de unos seis archivos PST por minuto con un tamaño medio de pocos megabytes. Por tanto, MRS es considerablemente más rápido que Enterprise Vault. Todo lo importado permanece en el buzón de diario hasta que el Task lo archiva y elimina. Por ello, `Resume-NICharge` vincula la liberación de más trabajos a la cantidad que aún no se ha movido, para que el buzón no crezca hasta el tamaño de toda la exportación.

## Script 2: Movimiento y ritmo

El segundo script se ejecuta como Vault Service Account en el servidor EV con Windows PowerShell 5.1. Mueve mensajes desde todas las subcarpetas de la bandeja de entrada de forma plana a la bandeja de entrada, excluye borradores y las carpetas *Versions* y *Konflikte*, y solo añade más cuando la bandeja de entrada está por debajo de un umbral.

```powershell
Add-Type -Path "C:\Program Files (x86)\Enterprise Vault\Microsoft.Exchange.WebServices.dll"
$NIProtokoll = "E:\pstimport\protokoll\verschieben-protokoll.csv"
$ews = New-Object Microsoft.Exchange.WebServices.Data.ExchangeService(
  [Microsoft.Exchange.WebServices.Data.ExchangeVersion]::Exchange2013_SP1)
$ews.UseDefaultCredentials = $true
$ews.Url = New-Object Uri("https://mailserver01.example.com/EWS/Exchange.asmx")
$mb = New-Object Microsoft.Exchange.WebServices.Data.Mailbox("journal@example.com")

function Schreibe-NIMove($Schritt, $Wert, $Meldung) {
  [pscustomobject]@{
    Zeit = Get-Date -Format "yyyy-MM-dd HH:mm:ss"; Schritt = $Schritt; Wert = $Wert; Meldung = $Meldung
  } | Export-Csv -Path $NIProtokoll -Append -NoTypeInformation -Encoding UTF8
}

function Get-NIInbox {
  [Microsoft.Exchange.WebServices.Data.Folder]::Bind($ews,
    (New-Object Microsoft.Exchange.WebServices.Data.FolderId(
      [Microsoft.Exchange.WebServices.Data.WellKnownFolderName]::Inbox, $mb)))
}

function Get-NIRueckstand {
  $inbox = Get-NIInbox
  $f = New-Object Microsoft.Exchange.WebServices.Data.SearchFilter+IsLessThan(
    [Microsoft.Exchange.WebServices.Data.ItemSchema]::DateTimeReceived, (Get-Date).AddHours(-2))
  $r = $inbox.FindItems($f, (New-Object Microsoft.Exchange.WebServices.Data.ItemView(1)))
  [pscustomobject]@{ Zeit = Get-Date -Format "HH:mm:ss"; Posteingang = $inbox.TotalCount; Alt = $r.TotalCount }
}

function Get-NIUnterordner {
  $inbox = Get-NIInbox
  $liste = New-Object System.Collections.ArrayList
  $offset = 0
  do {
    $fv = New-Object Microsoft.Exchange.WebServices.Data.FolderView(500, $offset)
    $fv.Traversal = [Microsoft.Exchange.WebServices.Data.FolderTraversal]::Deep
    $res = $inbox.FindFolders($fv)
    foreach ($f in $res.Folders) { $null = $liste.Add($f) }
    $offset += 500
  } while ($res.MoreAvailable)
  $ausgeschlossen = @("Versions", "Konflikte", "Conflicts", "Conflitti", "Conflits")
  $liste | Where-Object { $_.TotalCount -gt 0 -and $_.DisplayName -notin $ausgeschlossen }
}

function Move-NIPortion([int]$Max = 2000) {
  $inbox = Get-NIInbox
  $keinEntwurf = New-Object Microsoft.Exchange.WebServices.Data.SearchFilter+IsEqualTo(
    [Microsoft.Exchange.WebServices.Data.ItemSchema]::IsDraft, $false)
  $n = 0
  foreach ($ordner in Get-NIUnterordner) {
    $runden = 0
    do {
      $seite = $ordner.FindItems($keinEntwurf, (New-Object Microsoft.Exchange.WebServices.Data.ItemView(100)))
      if ($seite.Items.Count -gt 0) {
        $ids = New-Object 'System.Collections.Generic.List[Microsoft.Exchange.WebServices.Data.ItemId]'
        foreach ($i in $seite.Items) { $ids.Add($i.Id) }
        $antwort = $ews.MoveItems($ids, $inbox.Id)
        $fehler = @($antwort | Where-Object { $_.Result -ne "Success" }).Count
        if ($fehler -gt 0) { Schreibe-NIMove "Fehler" $fehler $ordner.DisplayName; break }
        $n += $seite.Items.Count
      }
      $runden++
    } while ($seite.Items.Count -gt 0 -and $n -lt $Max -and $runden -lt 200)
    if ($n -ge $Max) { break }
  }
  Schreibe-NIMove "Verschoben" $n ""
  return $n
}

function Start-NIDauerlauf([int]$Portion = 2000, [int]$MaxPosteingang = 3000) {
  $stop = "E:\pstimport\protokoll\STOP.txt"
  while (-not (Test-Path $stop)) {
    $s = Get-NIRueckstand
    if ($s.Posteingang -gt $MaxPosteingang) {
      "$($s.Zeit)  Posteingang $($s.Posteingang)  alt $($s.Alt)  warte auf EV"
      Schreibe-NIMove "Rueckstand" $s.Alt ("Posteingang " + $s.Posteingang)
      Start-Sleep -Seconds 300
      continue
    }
    $n = Move-NIPortion -Max $Portion
    "$(Get-Date -Format HH:mm:ss)  verschoben $n  (Posteingang vorher $($s.Posteingang))"
    if ($n -eq 0) { Start-Sleep -Seconds 600 }
  }
  "STOP-Datei gefunden, Dauerlauf beendet"
}

function Get-NIMoveStand {
  $p = Import-Csv $NIProtokoll
  "verschoben gesamt: " + (($p | Where-Object { $_.Schritt -eq "Verschoben" } | Measure-Object Wert -Sum).Sum)
  "Fehler: " + @($p | Where-Object { $_.Schritt -eq "Fehler" }).Count
  Get-NIRueckstand
}
```

<details class="options-details">
<summary>Explicación de las funciones</summary>

| Función | Efecto |
|---|---|
| `Get-NIRueckstand` | Número de elementos en la bandeja de entrada y número de elementos de más de dos horas; estos últimos son los reimportados, porque el flujo de diario en curso solo tiene horas de recepción actuales |
| `Get-NIUnterordner` | Todas las subcarpetas con contenido de la bandeja de entrada, excepto *Versions* y *Konflikte* |
| `Move-NIPortion -Max` | Mueve hasta N mensajes sin marca de borrador a la bandeja de entrada, en páginas de 100 mediante `MoveItems` con una lista tipada de `ItemId`; una matriz de PowerShell no coincide con la firma |
| `Start-NIDauerlauf -Portion -MaxPosteingang` | Bucle infinito: mide la bandeja de entrada cada cinco minutos y solo añade más si está por debajo del umbral; termina en cuanto existe el archivo `STOP.txt` |
| `Get-NIMoveStand` | Suma de elementos movidos, número de errores, retraso actual |

</details>

El umbral `MaxPosteingang` es la cifra más importante del procedimiento. Debe estar por encima de la carga de trabajo normal del Task (en el caso descrito, unos 1'500 informes a 100 mensajes por minuto y unos 20 minutos de retraso) y ser lo suficientemente bajo para que el Task nunca deje pendiente el flujo en curso. La siguiente sección muestra por qué es necesario.

## Métricas y regulación

Con la configuración predeterminada de Journaling Task (5 conexiones simultáneas al servidor Exchange, 1'000 elementos por pasada), Enterprise Vault archivó los mensajes reimportados durante la primera noche a un ritmo de unos 5'600 por hora. Por la mañana, las estadísticas de carpetas mostraron el coste: la bandeja de entrada tenía 16'500 en lugar de 1'500. El flujo de diario en curso, unos 6'000 informes por hora, llevaba más de dos horas de retraso. El Task había procesado también los mensajes antiguos y había dado prioridad a los informes actuales.

La capacidad del Task es la suma de ambos flujos. La reimportación solo puede recibir la parte que queda tras la actividad diaria. Por eso, el proceso continuo mide la propia bandeja de entrada, no el número de elementos antiguos, y solo añade más cuando el flujo está al día.

Tres mediciones muestran dónde está el límite del Task. Las colas entre Task y Storage Service son colas MSMQ; si las de Storage Service están vacías mientras las del Task están llenas, el Task está limitado al leer desde Exchange, no por el almacenamiento:

```powershell
Get-Counter -Counter '\MSMQ Queue(*)\Messages in Queue' -SampleInterval 5 -MaxSamples 6 |
  ForEach-Object {
    $t = $_.Timestamp.ToString("HH:mm:ss")
    $_.CounterSamples |
      Where-Object { $_.InstanceName -like "*enterprise vault*" -and $_.CookedValue -gt 0 } |
      ForEach-Object { "$t  $($_.InstanceName)  $($_.CookedValue)" }
  }
```

En Exchange, la latencia RPC se cuenta por tipo de cliente, no por número de solicitudes; valores inferiores a 10 ms indican que el servidor tiene reservas:

```powershell
$zaehler = @('\MSExchangeIS Client Type(*)\RPC Average Latency', '\Processor(_Total)\% Processor Time')
Get-Counter -ComputerName MAILSERVER01 -Counter $zaehler -SampleInterval 5 -MaxSamples 6 |
  ForEach-Object {
    $t = $_.Timestamp.ToString("HH:mm:ss")
    $_.CounterSamples | Where-Object { $_.CookedValue -gt 0 } |
      ForEach-Object { "$t  $($_.InstanceName)  $($_.Path.Split('\')[-1])  $([math]::Round($_.CookedValue,1))" }
  }
```

Y la directiva de limitación de Vault Service Account: Veritas exige una directiva con `RcaMaxConcurrency: Unlimited`; con la directiva estándar, Exchange limita las conexiones MAPI simultáneas por cuenta y no llegan conexiones adicionales del Task:

```powershell
Get-ThrottlingPolicyAssociation -Identity "svc-ev" | Format-List ThrottlingPolicyId
Get-ThrottlingPolicy | Format-Table Name, IsServiceAccount, RcaMaxConcurrency, EwsMaxConcurrency -AutoSize
```

Si el Task limita, el ajuste correspondiente se encuentra en las propiedades del Task (*Enterprise Vault Servers*, servidor, *Tasks*, Journaling Task, pestaña *Settings*):

| Ajuste | Predeterminado | Efecto |
|---|---|---|
| *Number of concurrent connections to Exchange Server* | 5 | Hilos que leen del buzón en paralelo; el efecto es lineal mientras la latencia de Exchange y las colas de Storage no presenten anomalías |
| *Maximum number of items per target per pass* | 1000 | Elementos por pasada; con mucho retraso, reinicia con menor frecuencia |

Los cambios se aplican tras reiniciar el Task (clic derecho, *Stop* y después *Start*). Conviene duplicar el valor y medir después mediante las marcas de tiempo del registro de movimiento, y luego pasar al siguiente nivel. No es posible tener un segundo Journaling Task para el mismo buzón; un buzón de diario está asignado a exactamente un Task. La paralelización real se logra con un segundo buzón de diario y un Task propio en un segundo servidor EV, en cuyo archivo se importa la segunda mitad de los archivos. Esto supone un cambio en la arquitectura de archivado y debe coordinarse con operaciones.

## Aspectos que llaman la atención

Una reimportación de esta magnitud revela cosas que antes nadie había revisado. Dos del caso descrito, porque probablemente se repitan.

La monitorización informó en el servidor Exchange de *RPC Requests/sec* por encima del umbral, con valores predeterminados de 60 y 70 solicitudes por segundo procedentes de la plantilla de comprobación. La carga base del servidor ya estaba por encima del umbral crítico sin importación. Las solicitudes por segundo son una cifra de carga; la cifra de salud es la latencia, que estaba por debajo de un milisegundo. El umbral debe redefinirse en función de la carga base del historial de monitorización, aproximadamente al doble y triple del pico diario habitual; el umbral de latencia se mantiene.


En la bandeja de entrada del buzón de diario había cuatro elementos desde hacía años que el Task volvía a intentar en cada pasada y rechazaba con el evento 3071. No habían aparecido en ningún informe porque nadie leía regularmente el registro de eventos `Veritas Enterprise Vault`. Lo mismo sucedía con el buzón de captura de `JournalingReportNdrTo`: 117'000 informes de diario no entregables desde 2020, una señal de que el archivado ya había perdido repetidamente informes antes de la brecha actual.

## Evidencia y cierre

Al final de la reimportación hay tres archivos y una búsqueda. `items.csv` de la exportación de Purview documenta lo que se exportó. `import-protokoll.csv` contiene por archivo lo que Exchange aceptó. `verschieben-protokoll.csv` contiene lo que se entregó al Task y cuándo. La búsqueda en el archivo de diario con el periodo de la brecha, desglosada por días, proporciona la cifra del archivo. Las diferencias entre estas cifras son explicables: borradores, elementos sin remitente, carpetas *Versions* y *Konflikte*, buzones fuera de retención. Precisamente esta explicación es la declaración para Compliance, junto con los dos límites que ninguna reimportación elimina: destinatarios de sobre ausentes y, sin deduplicación, copias en lugar de mensajes.

Después, la limpieza. Se eliminan del buzón de diario las subcarpetas vacías y los borradores que quedaron, se elimina el recurso compartido con los archivos PST y se borran los archivos PST en cuanto termina la conciliación. Se revoca la asignación del rol *Mailbox Import Export*; los ajustes del Task se mantienen si operaciones conoce las métricas. Y el propio incidente recibe una supervisión de la entrega de diario, pues la brecha se detectó por una alerta fortuita, no por monitorización.

## Fuentes

1.  [Microsoft Learn: Export search results in eDiscovery](https://learn.microsoft.com/en-us/purview/edisc-search-export): lista completa de las opciones de exportación de la nueva eDiscovery, tamaños de paquetes, comportamiento de *Organize data* e *Include folder and path*, caducidad de los paquetes a los 14 días.

2.  [Microsoft Learn: Deduplication in eDiscovery search results](https://learn.microsoft.com/en-us/purview/ediscovery-de-duplication-in-search-results): propiedades de comparación de la deduplicación clásica y la indicación de la retirada de las interfaces clásicas el 31 de agosto de 2025.

3.  [Microsoft Learn: Export items from a review set in eDiscovery](https://learn.microsoft.com/en-us/purview/edisc-review-set-export): la vía de exportación mediante un Review Set, que es la única en la nueva eDiscovery que excluye duplicados.

4.  [Microsoft Q&A: eDiscovery cases, deduplication on export](https://learn.microsoft.com/en-us/answers/questions/2201291/ediscovery-cases-deduplication-on-export): confirmación de que la exportación directa ya no ofrece deduplicación, con experiencias sobre el procedimiento mediante Review Set.

5.  [Microsoft Learn: New-MailboxImportRequest](https://learn.microsoft.com/en-us/powershell/module/exchange/new-mailboximportrequest): parámetros del trabajo de importación, requisitos para el recurso compartido y el rol *Mailbox Import Export*.

6.  [Microsoft Learn: Mailboxes are stalled during a migration](https://learn.microsoft.com/en-us/troubleshoot/exchange/migration/mailboxes-stalled-during-migration): gestión de carga de trabajo con diez trabajos simultáneos por destino y los estados `StalledDueToTarget` como comportamiento esperado.

7.  [Veritas: Enterprise Vault PST Migration, wizard-assisted migration](https://www.veritas.com/support/en_US/doc/95955885-161896939-0/v11744603-161896939): procedimiento del migrador PST, requisitos de acceso de Storage Service y tipos de archivo de destino permitidos.

8.  [Veritas VOX: Import PST to a Journal Archive](https://vox.veritas.com/t5/Enterprise-Vault/Import-PST-to-a-Journal-Archive/td-p/272928): experiencias de la comunidad sobre por qué el asistente no ofrece archivos de diario y cómo EVPM utiliza `ArchiveName`.
