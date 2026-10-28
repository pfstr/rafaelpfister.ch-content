---
title: "Cerrar una brecha de journaling: exportación desde Microsoft Purview e importación en Enterprise Vault"
navTitle: "Brecha de journaling"
description: "Si los informes de diario de Exchange Online fallan durante varios días, no se pueden generar posteriormente. Sin embargo, el contenido sigue estando en los buzones. Cómo exportarlo como PST mediante la nueva eDiscovery de Purview, por qué el migrador de PST de Enterprise Vault normalmente no es adecuado y cómo realizar la reimportación a través del buzón de diario sin desplazar el flujo de diario en curso, con dos scripts genéricos para importar, mover y registrar."
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
translationOf: journaling-luecke-purview-export-enterprise-vault-import
translationSourceHash: b22e076002b1662f409633c5b0b15584b7ef1d41cbe4d86f768096e0973f9d7c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-27T09:40:51.702Z
translationReview: automatic
aktualisiert: 2026-10-25
url: https://rafaelpfister.ch/es/blog/cerrar-una-brecha-de-journaling-exportar-desde-microsoft-purview-e-importar-en-enterprise-vault
---

# Cerrar una brecha de journaling: exportación desde Microsoft Purview e importación en Enterprise Vault

El journaling es un proceso de transporte. Exchange genera el informe de diario en el momento en que el mensaje pasa por el transporte y lo entrega al buzón de diario. Si esta entrega falla, por ejemplo porque un conector o un objeto de destinatario está configurado incorrectamente, no existe ninguna función que recupere el informe más tarde. Lo que pasó por el transporte durante ese período falta en el archivo.

Sin embargo, el contenido de los mensajes sigue estando en los buzones de los participantes y, con una directiva de retención sin expiración, también cuando los usuarios han eliminado los mensajes. De ello se deriva la única vía viable para completar el archivo: exportar el período desde todos los buzones e incorporar las copias al archivo de diario. El siguiente procedimiento utiliza la nueva eDiscovery de Microsoft Purview en el lado de la exportación y Veritas Enterprise Vault 15 en el lado de la importación; las mediciones proceden de una reimportación de varios cientos de miles de mensajes, durante la cual se crearon los dos scripts.

## Qué puede sustituir una exportación y qué no

Un informe de diario consta del sobre y del contenido. El sobre contiene los destinatarios a los que el transporte realmente entregó el mensaje: también las copias ocultas y los miembros resueltos de las listas de distribución. Una copia de buzón no contiene estos datos. Tras la reimportación ya no se puede determinar quién recibió un mensaje como copia oculta. En cambio, el contenido en sí puede reconstruirse por completo, siempre que se cumplan dos condiciones.

En primer lugar, debe haber una retención aplicada a los buzones, para que incluso los mensajes eliminados sigan estando en los elementos recuperables. Puede comprobarlo en PowerShell de Seguridad y Cumplimiento:

```powershell
Connect-IPPSSession
Get-RetentionCompliancePolicy -DistributionDetail |
  Format-List Name, Enabled, Mode, ExchangeLocation, ExchangeLocationException
Get-RetentionComplianceRule |
  Format-List Name, Policy, RetentionComplianceAction, RetentionDuration
```

Una directiva con `RetentionComplianceAction: Keep` y `RetentionDuration: Unlimited` mediante `ExchangeLocation: All` protege el contenido por completo. La lista de excepciones en `ExchangeLocationException` nombra los buzones para los que esto no se aplica. Para ellos, solo puede reconstruirse lo que aún esté disponible.

En segundo lugar, una exportación cuenta copias de buzón, no mensajes. Un mensaje dirigido a quince destinatarios internos aparece quince veces en el resultado. En el caso descrito, para un solo día había 98'131 copias frente a unas 6'800 mensajes únicos, determinados a partir del seguimiento de mensajes. El tratamiento de estas copias lo decide Compliance, no TI (véase la sección sobre deduplicación).

## Determinar el período

El inicio de la brecha se encuentra en el propio buzón de diario, en el servidor Exchange que lo aloja. La hora del último informe entregado marca el inicio:

```powershell
Get-MailboxFolderStatistics "journal@example.com" -IncludeOldestAndNewestItems |
  Where-Object { $_.ItemsInFolder -gt 0 } |
  Format-Table Name, ItemsInFolder, OldestItemReceivedDate, NewestItemReceivedDate -AutoSize
```

Lo mismo se aplica al buzón que figura en `JournalingReportNdrTo` de la configuración de transporte. Este recopila informes de diario no entregables, y su entrada más reciente confirma el momento. El final de la brecha es el momento en que la regla de diario se cambió al destino reparado. Entre ambos se encuentra el período de exportación, en UTC.

El seguimiento de mensajes proporciona la cifra esperada para la comparación: una línea por destinatario y mensaje, con `MessageId`. Un informe histórico cubre todo el período:

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

Los valores únicos de `MessageId` de este informe son la cifra con la que se medirá la reimportación al final.

## Exportación desde la nueva eDiscovery

Microsoft desactivó las interfaces clásicas de Content Search y eDiscovery (Standard) el 31 de agosto de 2025. Lo que aún aparece en muchas guías, en particular la opción de deduplicación durante la exportación, ya no existe en la nueva eDiscovery. El proceso en el portal de Purview:

1. En *eDiscovery*, cree o abra un caso. Para exportar, la cuenta ejecutora necesita el rol *eDiscovery Manager* y una licencia Microsoft 365 E3 o E5.
2. Cree una búsqueda. Fuente de datos: todos los buzones o, para una prueba, un único buzón.
3. Utilice KQL como consulta. Los paréntesis son decisivos porque `AND` tiene mayor precedencia que `OR`:

```text
kind:email AND ((received>=2026-09-10 AND received<2026-09-12)
             OR (sent>=2026-09-10 AND sent<2026-09-12))
```

Sin `kind:email` la búsqueda también cuenta entradas de calendario, contactos y tareas. En el caso descrito, esto marcó la diferencia entre 473'725 y 98'131 resultados para un día. Sin los paréntesis exteriores, la segunda condición de fecha se aplica solo a `sent`, y el resultado contiene mensajes de todo el contenido del buzón.

4. Ejecute *Generate statistics*. En búsquedas de todo el tenant, esto acelera considerablemente la exportación, porque solo se procesan los buzones con resultados. Antes de exportar, procese las ubicaciones que terminen con error mediante *Retry failed locations*; de lo contrario, faltarán en el resultado y eso solo se detectará tras la descarga.
5. Seleccione *Export* con la siguiente configuración.

| Opción | Efecto |
|---|---|
| *Export type: Export items with items report* | Mensajes más `items.csv`; el informe sin elementos no contiene contenido |
| *Export format: Create PSTs for messages* | Un PST por buzón en lugar de archivos `.msg` individuales |
| *Maximum PST package size* | A partir de este tamaño, un PST se divide en partes (`.001.pst`, `.002.pst`) |
| *Maximum .zip package size* | Tamaño de los paquetes de descarga; debe ser al menos igual al tamaño del PST |
| *Organize data from different locations into separate folders or PSTs* | **Activar**: un archivo por buzón, llamado `<smtp-adresse>.001.pst` |
| *Include folder and path of the source* | **Desactivar**: todos los mensajes terminan en la carpeta `Items` del PST en lugar de en la estructura de carpetas del buzón |
| *Give each item a friendly name* | Sin efecto en la exportación PST |

La configuración *Include folder and path of the source* determina la importación posterior. Con estructura de carpetas, la importación crea en el buzón de diario las carpetas de cada buzón, en el idioma de cada usuario: `Posteingang`, `Posta in arrivo`, `Boîte de réception`. Sin estructura de carpetas, todo queda en una carpeta `Items`, y la reimportación dispone de un único lugar desde el que continuar.

La descarga proporciona paquetes Zip. Microsoft recomienda 7-Zip o una herramienta comparable en lugar del Explorador de Windows. Los paquetes caducan 14 días después de su creación. El archivo `items.csv` del informe de proceso de la exportación (en *Process manager*, Export, Informes) acredita qué contiene la exportación y qué se omitió; forma parte del conjunto de evidencias.

### Deduplicación

La Content Search clásica disponía de la casilla *Enable de-duplication*: los mensajes con el mismo `InternetMessageId`, `ConversationTopic` y `BodyTagInfo` solo se exportaban una vez; las demás ubicaciones se indicaban en `Results.csv`. La nueva eDiscovery no ofrece esta posibilidad en la exportación directa desde una búsqueda. La única vía documentada es un *Review Set*: cargar allí los resultados de búsqueda, ejecutar Analytics, utilizar el filtro generado automáticamente *For Review*, que excluye duplicados, y exportar desde el Review Set. Esto requiere eDiscovery Premium y, por tanto, una licencia E5 para el usuario ejecutor. Varios usuarios informan en Microsoft Q&A de duplicados pese a este procedimiento; es aconsejable hacer una prueba con un día antes de la exportación completa.

Sin deduplicación se aplica lo siguiente: Enterprise Vault deduplica en el nivel de almacenamiento (los adjuntos y cuerpos de mensaje idénticos se almacenan una sola vez), pero no en el nivel de elementos. Cada copia de buzón se convierte en una entrada de archivo independiente, un resultado de búsqueda independiente y un contador independiente. Por tanto, el recuento para la evidencia no puede derivarse del archivo, sino solo de `items.csv` frente al seguimiento de mensajes. Esta decisión, aceptar copias o volver a exportar mediante un Review Set, se toma antes de la importación, no después.

## Las vías hacia Enterprise Vault

Enterprise Vault incluye el migrador de PST para archivos PST, como asistente en la consola de administración (*Archives*, clic derecho, *Import PST*) y como variante scriptable mediante el Policy Manager EVPM. Para la reimportación a un archivo de diario, esta vía presenta tres limitaciones.

Primero, la licencia. El migrador requiere la funcionalidad `EVPSTM` (*Exchange PST Migrator*). En un sitio que solo opera journaling, a menudo no está licenciada, y el asistente se interrumpe en la primera página con *Required license not installed*. El Policy Manager utiliza el mismo migrador y muestra el mismo mensaje.

Segundo, el destino. El asistente no ofrece archivos de diario como destino, solo archivos de buzón y archivos de correo de Internet. La importación directa al archivo de diario solo es posible mediante EVPM con `ArchiveName`, y con ello se omiten las reglas de procesamiento del Journaling Task. Como destino queda un Shared Archive propio en el Vault Store del diario, que la búsqueda cubre junto con el archivo de diario.

Tercero, la trazabilidad: el migrador escribe en el archivo PST (marca los elementos migrados), por lo que el conjunto se modifica, y los informes de migración de cada archivo deben compararse con las cifras de exportación.

La segunda vía no requiere licencia adicional: el Exchange Journaling Task de Enterprise Vault no archiva únicamente informes de diario. Si reconoce un elemento como informe de diario, lo desempaqueta y adopta los datos del sobre. Archiva directamente un elemento ordinario, como siempre hizo en tiempos del journaling simple de Exchange sin sobre. Quien importe los archivos PST en el buzón de diario consigue que el Task habitual escriba los mensajes en el archivo de diario habitual, con la misma categoría de retención, la misma indexación y sin cambios en la arquitectura de archivado.

Esto puede demostrarse con un único mensaje de prueba al buzón de diario: si poco después ya no está en ninguna carpeta, tampoco en *Invalid Journal Report*, y la búsqueda lo encuentra en el archivo de diario, el Task archiva mensajes ordinarios.

Esta vía tiene dos características que determinan el procedimiento. El Task procesa exclusivamente el nivel superior de la bandeja de entrada, no las subcarpetas. Puede comprobarse en el buzón de diario: la carpeta de búsqueda *Initial Trawl With Pending* en *Enterprise Vault Search Folders* cuenta exactamente los elementos de la bandeja de entrada, sin incluir subcarpetas. Y la importación mediante Exchange siempre crea carpetas; cómo lo hace se explica en la siguiente sección.

## Importación en el buzón de diario

`New-MailboxImportRequest` importa un archivo PST del lado del servidor mediante el Mailbox Replication Service. El archivo debe encontrarse en un recurso compartido sobre el que *Exchange Trusted Subsystem* tenga control total, y la cuenta ejecutora necesita el rol *Mailbox Import Export*, que no se asigna a nadie de forma predeterminada:

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
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Mailbox` | Buzón de destino, aquí el buzón de diario |
| `-FilePath` | Ruta UNC del archivo PST; se rechazan las rutas locales |
| `-TargetRootFolder` | Carpeta bajo la cual se deposita el contenido PST; si no se indica, las carpetas PST se asignan a carpetas de buzón con el mismo nombre |
| `-SourceRootFolder` | Carpeta del PST a partir de la cual se importa; en la práctica, la propia carpeta se crea de todos modos (véase el texto) |
| `-Name` | Nombre único del trabajo; para miles de archivos se deriva del nombre de archivo |
| `-BatchName` | Agrupación para consultas y estadísticas |

</details>

Tres observaciones de la prueba determinan el resto del procedimiento. Sin `TargetRootFolder` y con un PST estructurado (estructura de carpetas exportada), MRS crea la raíz del PST como una carpeta independiente junto a la bandeja de entrada; en el caso de un buzón alemán, `Oberste-Ebene-des-Informationsspeichers` con las subcarpetas `Posteingang`, `Gesendete-Elemente` y otras, además de `Recoverable-Items` con `Deletions`. Con `TargetRootFolder "Inbox"` y un PST plano, todos los mensajes terminan en `/Inbox/Items`. Y `SourceRootFolder "Items"` combinado con `TargetRootFolder "Inbox"` también produjo en la prueba `/Inbox/Items`; el nivel no se eliminó.

En los tres casos, los mensajes quedan fuera de la vista del Journaling Task. El último paso, moverlos planos a la bandeja de entrada, no puede realizarse con herramientas nativas de Exchange; se ejecuta mediante EWS. La EWS Managed API se encuentra como `Microsoft.Exchange.WebServices.dll` en el directorio de instalación de Enterprise Vault, y la Vault Service Account tiene control total sobre el buzón de diario, por lo que no se requiere ningún permiso adicional. El script se ejecuta en el servidor EV con Windows PowerShell 5.1, ya que la biblioteca se basa en .NET Framework.

Este paso de movimiento es al mismo tiempo el regulador de todo el procedimiento: decide cuántos mensajes ve el Task de una vez.

## La prueba con un buzón

Antes de procesar miles de archivos, puede verificarse todo el proceso con un único buzón. Resulta útil el buzón de la persona que realizó la exportación: tiene la autorización, son sus propios datos y el archivo es pequeño. El procedimiento:

1. Copia del archivo PST en una carpeta propia, hash SHA-256 del original en un archivo CSV.
2. Importación con `TargetRootFolder "Inbox"`, estadística con `Get-MailboxImportRequestStatistics`: `ItemsTransferred` es la cifra esperada.
3. Estadísticas de carpetas del buzón de diario: ¿dónde están los elementos?
4. Mover desde `/Inbox/Items` a la bandeja de entrada mediante EWS.
5. Recuento de los elementos antiguos de la bandeja de entrada cada pocos minutos: los elementos con `DateTimeReceived` anterior al inicio del flujo en curso son los importados. Si el número baja, el Task está archivando.
6. Comparación: búsqueda en el archivo de diario (en Discovery Accelerator o en la búsqueda de EV) con período y propietario del buzón; número de resultados frente a `ItemsTransferred`.

En el caso descrito, se importaron 128 elementos y se archivaron 126. Los dos restantes eran un borrador y un elemento sin remitente; el Task deja los elementos sin remitente. Los borradores nunca pasaron por el transporte y no pertenecen a ningún diario, por lo que el script de movimiento los excluye mediante la marca `IsDraft`, independientemente del idioma. Tampoco se mueven los elementos de las carpetas *Versions* y *Konflikte* (versiones anteriores bajo retención, restos de sincronización).

El registro de eventos `Veritas Enterprise Vault` muestra después de la prueba lo que el Task rechazó. El evento 3071 (*could not be archived as it may be corrupt*) y el 3288 (*no longer archive pending*) se repiten en cada pasada para los mismos elementos; dichos elementos deben moverse a una carpeta fuera de la bandeja de entrada para que el Task no vuelva a intentarlo en cada pasada.

## Script 1: importación por lotes

El primer script se ejecuta en Exchange Management Shell en un servidor Exchange. Crea trabajos de importación por lotes, espera, registra para cada archivo el estado y `ItemsTransferred` en un archivo CSV, y elimina los trabajos completados. Las ejecuciones repetidas omiten los archivos que ya figuran en el registro. El prefijo `NI` significa reimportación.

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
<summary>Funciones explicadas</summary>

| Función | Efecto |
|---|---|
| `Get-NIDateien` | Lista todos los archivos PST de las carpetas de origen con un nombre de trabajo derivado; los mismos nombres de archivo en carpetas distintas (por ejemplo, una exportación por día) siguen siendo distinguibles |
| `Start-NICharge -Anzahl` | Crea hasta N trabajos de importación para archivos aún no registrados |
| `Wait-NICharge` | Espera hasta que no haya ningún trabajo en ejecución, escribe el estado y `ItemsTransferred` en el registro, elimina los trabajos completados; los fallidos permanecen para su análisis. Las estadísticas y la eliminación se ejecutan mediante la canalización porque `-Identity` no acepta el objeto Identity deserializado en una shell remota |
| `Resume-NICharge -Anzahl -MaxLager` | Solo libera trabajos suspendidos cuando hay menos de `MaxLager` mensajes pendientes en las subcarpetas de la bandeja de entrada |
| `Get-NIImportStand` | Resumen por estado, suma de los elementos importados y número total de archivos |

</details>

Debe conocer dos comportamientos de MRS. Exchange permite de forma predeterminada diez trabajos simultáneos en el mismo buzón de destino; todos los demás quedan en la cola con un motivo como `StalledDueToTarget_MailboxCapacityExceeded` o `StalledDueToTarget_MdbReplication` y avanzan después. Es gestión de carga de trabajo, no un problema de capacidad, aunque el nombre lo sugiera. Además, `Get-MailboxImportRequest` muestra el estado con retraso; las estadísticas son más actuales.

El rendimiento fue de unos seis archivos PST por minuto con un tamaño medio de pocos megabytes. Por tanto, MRS es considerablemente más rápido que Enterprise Vault. Todo lo que se ha importado permanece en el buzón de diario hasta que el Task lo archiva y elimina. Por eso, `Resume-NICharge` vincula la liberación de más trabajos a la cantidad que aún no se ha movido, para que el buzón no crezca hasta el tamaño de toda la exportación.

## Script 2: mover y marcar el ritmo

El segundo script se ejecuta como Vault Service Account en el servidor EV con Windows PowerShell 5.1. Mueve los mensajes de todas las subcarpetas de la bandeja de entrada, de forma plana, a la bandeja de entrada; excluye los borradores y las carpetas *Versions* y *Konflikte*, y solo añade más cuando la bandeja de entrada está por debajo de un umbral.

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
<summary>Funciones explicadas</summary>

| Función | Efecto |
|---|---|
| `Get-NIRueckstand` | Número de elementos en la bandeja de entrada y número de elementos con más de dos horas; estos últimos son los reimportados, ya que el flujo de diario en curso solo tiene horas de recepción actuales |
| `Get-NIUnterordner` | Todas las subcarpetas con contenido de la bandeja de entrada, sin *Versions* ni *Konflikte* |
| `Move-NIPortion -Max` | Mueve hasta N mensajes sin marca de borrador a la bandeja de entrada, en páginas de 100 mediante `MoveItems` con una lista tipada de `ItemId`; una matriz de PowerShell no se ajusta a la firma |
| `Start-NIDauerlauf -Portion -MaxPosteingang` | Bucle infinito: mide la bandeja de entrada cada cinco minutos y solo añade más si está por debajo del umbral; termina cuando existe el archivo `STOP.txt` |
| `Get-NIMoveStand` | Suma de los elementos movidos, número de errores y retraso actual |

</details>

El umbral `MaxPosteingang` es la cifra más importante del procedimiento. Debe situarse por encima de la carga de trabajo normal del Task (en el caso descrito, unos 1'500 informes con 100 mensajes por minuto y aproximadamente 20 minutos de retraso) y ser lo suficientemente bajo para que el Task nunca deje pendiente el flujo en curso. La siguiente sección explica por qué es necesario.

## Mediciones y el regulador

Con la configuración predeterminada del Journaling Task (5 conexiones simultáneas al servidor Exchange, 1'000 elementos por pasada), Enterprise Vault archivó los mensajes reimportados durante la primera noche a un ritmo de unos 5'600 por hora. Por la mañana, las estadísticas de carpetas mostraron el coste: la bandeja de entrada estaba en 16'500 en lugar de 1'500. El flujo de diario en curso, de unos 6'000 informes por hora, llevaba más de dos horas de retraso. El Task había procesado también los mensajes antiguos y dado prioridad a los informes actuales.

La capacidad del Task es la suma de ambos flujos. La reimportación solo puede recibir la parte que queda tras la actividad diaria. Por eso, la ejecución continua mide la propia bandeja de entrada, no la cantidad de elementos antiguos, y solo añade más cuando el flujo está al día.

Tres mediciones muestran dónde está el límite del Task. Las colas entre el Task y el Storage Service son colas MSMQ; si las del Storage Service están vacías mientras las del Task están llenas, el Task se frena al leer desde Exchange, no el almacenamiento:

```powershell
Get-Counter -Counter '\MSMQ Queue(*)\Messages in Queue' -SampleInterval 5 -MaxSamples 6 |
  ForEach-Object {
    $t = $_.Timestamp.ToString("HH:mm:ss")
    $_.CounterSamples |
      Where-Object { $_.InstanceName -like "*enterprise vault*" -and $_.CookedValue -gt 0 } |
      ForEach-Object { "$t  $($_.InstanceName)  $($_.CookedValue)" }
  }
```

En el lado de Exchange, la latencia RPC se cuenta por tipo de cliente, no el número de solicitudes; valores inferiores a 10 ms indican que el servidor tiene margen:

```powershell
$zaehler = @('\MSExchangeIS Client Type(*)\RPC Average Latency', '\Processor(_Total)\% Processor Time')
Get-Counter -ComputerName MAILSERVER01 -Counter $zaehler -SampleInterval 5 -MaxSamples 6 |
  ForEach-Object {
    $t = $_.Timestamp.ToString("HH:mm:ss")
    $_.CounterSamples | Where-Object { $_.CookedValue -gt 0 } |
      ForEach-Object { "$t  $($_.InstanceName)  $($_.Path.Split('\')[-1])  $([math]::Round($_.CookedValue,1))" }
  }
```

Y la directiva de limitación de la Vault Service Account: Veritas requiere una directiva con `RcaMaxConcurrency: Unlimited`; con la directiva estándar, Exchange limita las conexiones MAPI simultáneas por cuenta, y las conexiones adicionales del Task no llegan:

```powershell
Get-ThrottlingPolicyAssociation -Identity "svc-ev" | Format-List ThrottlingPolicyId
Get-ThrottlingPolicy | Format-Table Name, IsServiceAccount, RcaMaxConcurrency, EwsMaxConcurrency -AutoSize
```

Si el Task se ralentiza, la configuración adecuada está en las propiedades del Task (*Enterprise Vault Servers*, servidor, *Tasks*, Journaling Task, pestaña *Settings*):

| Configuración | Predeterminado | Efecto |
|---|---|---|
| *Number of concurrent connections to Exchange Server* | 5 | Hilos que leen en paralelo desde el buzón; el efecto es lineal mientras la latencia de Exchange y las colas de Storage no muestren problemas |
| *Maximum number of items per target per pass* | 1000 | Elementos por pasada; con gran retraso, reinicia con menor frecuencia |

Los cambios se aplican tras reiniciar el Task (clic derecho, *Stop* y luego *Start*). Tiene sentido duplicar el valor y medir después mediante las marcas de tiempo del registro de movimientos, y luego pasar al siguiente nivel. No es posible un segundo Journaling Task para el mismo buzón; un buzón de diario está asignado exactamente a un Task. La paralelización real se consigue con un segundo buzón de diario y un Task propio en un segundo servidor EV, en cuyo archivo se importa la segunda mitad de los archivos. Esto modifica la arquitectura de archivado y debe coordinarse con Operaciones.

## Aspectos que llaman la atención en los márgenes

Una reimportación de esta magnitud revela cosas que antes nadie había revisado. Dos de ellas proceden del caso descrito, porque probablemente se repetirán.

El monitoreo alertaba en el servidor Exchange de *RPC Requests/sec* por encima del umbral, con valores predefinidos de 60 y 70 solicitudes por segundo procedentes de la plantilla de comprobación. La carga base del servidor ya superaba el umbral crítico sin importación. Las solicitudes por segundo son una cifra de carga; la cifra de salud es la latencia, y esta era inferior a un milisegundo. El umbral debe reajustarse según la carga base del historial de monitoreo, por ejemplo al doble y al triple del pico diurno habitual; el umbral de latencia se mantiene.


En la bandeja de entrada del buzón de diario había desde hacía años cuatro elementos que el Task volvía a intentar en cada pasada y rechazaba con el evento 3071. No habían aparecido en ningún informe porque nadie leía regularmente el registro de eventos `Veritas Enterprise Vault`. Lo mismo ocurría con el buzón de captura de `JournalingReportNdrTo`: 117'000 informes de diario no entregables desde 2020, indicio de que el archivado ya había perdido informes repetidamente antes de la brecha actual.

## Evidencia y cierre

Al final de la reimportación hay tres archivos y una búsqueda. `items.csv` de la exportación de Purview acredita lo que se exportó. `import-protokoll.csv` contiene, por archivo, lo que Exchange incorporó. `verschieben-protokoll.csv` contiene lo que se entregó al Task y cuándo. La búsqueda en el archivo de diario con el período de la brecha, dividida por días, proporciona la cifra del archivo. Las diferencias entre estas cifras son explicables: borradores, elementos sin remitente, carpetas *Versions* y *Konflikte*, buzones fuera de la retención. Esta explicación es precisamente la declaración para Compliance, junto con los dos límites que ninguna reimportación elimina: destinatarios del sobre ausentes y, sin deduplicación, copias en vez de mensajes.

Después viene la limpieza. Se eliminan del buzón de diario las subcarpetas vacías y los borradores que hayan quedado; se elimina el recurso compartido con los archivos PST y se borran los archivos PST en cuanto finalice la comparación. Se revoca la asignación del rol *Mailbox Import Export*; la configuración del Task se mantiene si Operaciones conoce las mediciones. Y el propio incidente recibe monitoreo de la entrega de diario, ya que la brecha se detectó mediante una alerta casual, no mediante monitoreo.

## Historial de cambios

- **25 de octubre de 2026:** Se sustituyó la fuente de la comunidad sobre la importación de PST en archivos de diario por un artículo de soporte de Enterprise Vault que confirma la falta de compatibilidad y menciona Shared Archive o la copia al buzón de diario como alternativas.

## Fuentes

1.  [Microsoft Learn: Export search results in eDiscovery](https://learn.microsoft.com/en-us/purview/edisc-search-export): lista completa de las opciones de exportación de la nueva eDiscovery, tamaños de paquete, comportamiento de *Organize data* e *Include folder and path*, caducidad de los paquetes a los 14 días.

2.  [Microsoft Learn: Deduplication in eDiscovery search results](https://learn.microsoft.com/en-us/purview/ediscovery-de-duplication-in-search-results): propiedades de comparación de la deduplicación clásica y nota sobre la desactivación de las interfaces clásicas el 31 de agosto de 2025.

3.  [Microsoft Learn: Export items from a review set in eDiscovery](https://learn.microsoft.com/en-us/purview/edisc-review-set-export): la vía de exportación mediante un Review Set, que es la única que excluye duplicados en la nueva eDiscovery.

4.  [Microsoft Q&A: eDiscovery cases, deduplication on export](https://learn.microsoft.com/en-us/answers/questions/2201291/ediscovery-cases-deduplication-on-export): confirmación de que la exportación directa ya no ofrece deduplicación, con experiencias sobre la vía de Review Set.

5.  [Microsoft Learn: New-MailboxImportRequest](https://learn.microsoft.com/en-us/powershell/module/exchange/new-mailboximportrequest): parámetros del trabajo de importación, requisitos para el recurso compartido y el rol *Mailbox Import Export*.

6.  [Microsoft Learn: Mailboxes are stalled during a migration](https://learn.microsoft.com/en-us/troubleshoot/exchange/migration/mailboxes-stalled-during-migration): gestión de carga de trabajo con diez trabajos simultáneos por destino y los estados `StalledDueToTarget` como comportamiento esperado.

7.  [Veritas: Enterprise Vault PST Migration, wizard-assisted migration](https://www.veritas.com/support/en_US/doc/95955885-161896939-0/v11744603-161896939): proceso del migrador de PST, requisitos de acceso del Storage Service y tipos de archivo de destino permitidos.

8.  [Enterprise Vault Support: Unable to migrate a Personal Store (PST) into a Journal Archive](https://supportenterprisevault.cloud.com/external/article/100029869/unable-to-migrate-a-personal-store-pst-i.html): la importación de PST a un archivo de diario no es compatible; el artículo menciona como alternativas un Shared Archive o copiar los mensajes a la bandeja de entrada del buzón de diario.
