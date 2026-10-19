---
title: "Caídas de Exchange tras las actualizaciones de Windows Server de octubre de 2026: MapiExceptionMdbOffline (ec=1142)"
navTitle: "Caída de Exchange 10/2026"
description: "Tras las actualizaciones de Windows Server del 13 de octubre de 2026, los servidores Exchange notifican MapiExceptionMdbOffline (ec=1142) al cabo de unas 24 horas: OWA falla, las colas crecen y las copias DAG dejan de replicarse. La causa es un nuevo límite en el RPC Endpoint Mapper; desde el 17 de octubre de 2026, Microsoft recomienda el valor de registro EpmapSavedContextCap, pero aún no hay una corrección. Síntomas, KB afectados, comandos de comprobación y procedimiento."
date: "2026-10-16"
kategorie: "Exchange OnPrem / Híbrido"
timeToRead: "10 min de lectura"
themen:
  - exchange-updates
  - exchange-onprem-hybrid
produkte:
  - "exchange-updates"
  - "exchange-on-premises"
protokolle:
  - "troubleshooting"
  - "releases"
  - "powershell"
slug: "caidas-de-exchange-tras-las-actualizaciones-de-windows-server-de-octubre-de-2026"
featured: "2026-10-16"
warnung: true
translationId: "article-8fdf2b2744bdbd41"
translationOf: exchange-windows-updates-oktober-2026
translationSourceHash: 47c39c05826230354e6b431330725587440880dac2b6a33c00f5d34da2afe2d7
translationModel: gpt-5.6-terra
translatedAt: 2026-10-19T07:50:40.721Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/caidas-de-exchange-tras-las-actualizaciones-de-windows-server-de-octubre-de-2026
---

# Caídas de Exchange tras las actualizaciones de Windows Server de octubre de 2026: MapiExceptionMdbOffline (ec=1142)

Desde el Patch Tuesday del 13 de octubre de 2026, los servidores Exchange dejan de funcionar tras unas 24 horas de actividad: los clientes pierden la conexión con el buzón, OWA muestra `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`, los correos permanecen en las colas y las copias de bases de datos en DAG dejan de replicarse. Microsoft añadió una nota al anuncio de las SU de Exchange de octubre el 15 de octubre de 2026: el desencadenante son las actualizaciones de octubre para Windows Server; la instalación de la SU de Exchange no es necesaria para que se produzca el problema. Desde el 17 de octubre de 2026 se conocen la causa y una medida oficial mediante el registro; aún no existe una corrección. Los detalles se indican en la siguiente sección.

## Actualización del 19 de octubre de 2026: causa y medida mediante el registro

El equipo de Exchange publicó una entrada específica sobre el problema el 17 de octubre de 2026. En las notas de la versión de las actualizaciones de Windows (por ejemplo, KB5126646), el problema aparece ahora como problema conocido con el estado «sin resolver».

### Causa

Las actualizaciones de octubre limitan la memoria que RPC Endpoint Mapper utiliza para las llamadas de búsqueda de endpoints. De forma predeterminada, ahora solo se permiten 512 contextos almacenados; según Microsoft, esto no basta para cargas como Exchange. Exchange acumula estos contextos durante el funcionamiento hasta alcanzar el límite. Después, las conexiones RPC a la base de datos de buzones fallan y OWA muestra `MapiExceptionMdbOffline`. Un lector de Born ha demostrado el nuevo valor de configuración `EpmapSavedContextCap` en la `RpcEpMap.dll` de las actualizaciones de octubre. Frankys Web atribuye el límite a la vulnerabilidad de denegación de servicio CVE-2026-96300 en RPC Endpoint Mapper, corregida el mismo Patch Tuesday.

Esto explica varias observaciones de los primeros días:

- La caída solo se produce tras varias horas y el momento depende de la carga. Según Microsoft, algunos servidores nunca se ven afectados.
- Reiniciar el servidor restablece el contador y, por tanto, solo ayuda hasta que se vuelve a alcanzar el límite. Según un lector de Born, reiniciar únicamente los servicios de Exchange no libera la memoria ocupada.
- El límite está incluido en las actualizaciones acumulativas de Windows. La SU de Exchange y la actualización de .NET no intervienen.
- Por tanto, la presunta relación con las tareas de mantenimiento de bases de datos y el servicio de replicación no es la causa.

### Medida correctiva

Microsoft recomienda aumentar el límite mediante el registro y reiniciar el servidor después. En una ventana de comandos con privilegios de administrador:

```cmd
reg add "HKLM\SOFTWARE\Microsoft\Rpc" /v EpmapSavedContextCap /t REG_DWORD /d 65536 /f
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `"HKLM\SOFTWARE\Microsoft\Rpc"` | Clave del registro de la configuración RPC. |
| `/v EpmapSavedContextCap` | Nombre del valor que establece el límite de contextos almacenados de Endpoint Mapper (sin entrada se aplica 512). |
| `/t REG_DWORD` | Tipo de datos: número de 32 bits. |
| `/d 65536` | Nuevo límite, según recomienda Microsoft. |
| `/f` | Sobrescribe un valor existente sin solicitar confirmación. |

</details>

Endpoint Mapper solo lee el valor al iniciarse; sin reiniciar el servidor, el cambio no surtirá efecto. Si el problema vuelve a aparecer, Microsoft recomienda aumentar más el valor. Según Microsoft, la medida ha resuelto el problema en los casos de clientes atendidos hasta ahora. Según Born, Nino Bilic (Microsoft) describe la entrada como una solución temporal que no supone un riesgo de seguridad, pero que no es la solución definitiva. Un lector de Born informa de un servidor que lleva más de 26 horas funcionando sin interrupciones con 65536. Algunos administradores utilizan valores más altos, como 16777215; hasta ahora no se ha documentado ningún caso en el que 65536 no haya sido suficiente tras un reinicio.

Esta consulta en PowerShell muestra si el valor está establecido:

```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Rpc" -Name EpmapSavedContextCap
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-Path "HKLM:\SOFTWARE\Microsoft\Rpc"` | Clave del registro donde se encuentra el valor. |
| `-Name EpmapSavedContextCap` | Lee solo este valor; si falta, PowerShell muestra un error y se aplica el límite predeterminado de 512. |

</details>

Establezca el valor en todos los servidores Exchange, incluidos aquellos que todavía no hayan fallado. En servidores que aún no tengan las actualizaciones de octubre, puede establecer el valor antes de instalarlas; el reinicio posterior a la instalación lo activará. En una DAG, reinicie los miembros uno tras otro como se describe a continuación. Frankys Web también proporciona un script de PowerShell que establece el valor, guarda el estado anterior y puede revertirse con `-Remove`.

Según Microsoft, las actualizaciones de Windows no deben desinstalarse expresamente. Un lector de Frankys Web también informa de problemas de inicio de sesión RDP (CredSSP) después de desinstalar KB5126868. Si la medida del registro no basta, Microsoft ofrece otra medida a través de Support for business. En cuanto aparezca una corrección, deberá eliminar de nuevo el valor del registro, porque el límite más alto debilita la protección frente a CVE-2026-96300.

### Cuestiones pendientes

- **Corrección**: Microsoft no indica ninguna fecha. Un lector de Born supone que Microsoft podría retirar el límite mediante un Known Issue Rollback (Feature-ID 63096240), pero no está confirmado.
- **Valor**: No está claro si 65536 también bastará en entornos grandes hasta el próximo Patch Tuesday. Por ello, siga supervisando el Event ID 1006.
- **Otros roles de servidor**: Un lector de Born también observa servidores sin Exchange afectados. Microsoft solo se pronuncia sobre Exchange.
- **Copia de seguridad**: No está claro si el error notificado de VSS Writer es una consecuencia de la caída.

## Síntomas

Los síntomas comunicados hasta ahora:

- OWA falla con `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`; el registro Application contiene el Event ID 1006.
- Los clientes Outlook pierden la conexión con el buzón.
- Los correos no se entregan ni interna ni externamente, y las colas de transporte crecen.
- En DAG, las copias de bases de datos dejan de replicarse; según informes de lectores de Born, los switchover al pasar al modo de mantenimiento fallan.
- Exchange Admin Center sigue mostrando las bases de datos como montadas (Frankys Web).

El error no se produce inmediatamente después de la instalación, sino, según los informes, tras unas 22 a 30 horas de funcionamiento. Reiniciar el servidor restablece el funcionamiento, pero la caída se repite aproximadamente al cabo del mismo tiempo. En comentarios de Born se observa que el servicio de replicación de buzones se detiene durante o después de las tareas de mantenimiento de bases de datos y que posteriormente comienzan los errores MdbOffline; según la causa conocida ahora, esto es un efecto secundario, no el desencadenante.

## Actualizaciones afectadas

Microsoft menciona estas actualizaciones acumulativas de Windows en la adenda al anuncio de Exchange:

| Windows Server | Actualización |
| --- | --- |
| Windows Server 2025 | KB5126646 |
| Windows Server 2022 | KB5126655 |
| Windows Server 2019 | KB5126651 |
| Windows Server 2016 | KB5126868 |

El mismo Patch Tuesday también se publicaron las SU de Exchange (Exchange SE: KB5128895, «SU11») y actualizaciones acumulativas para .NET Framework. En un Exchange SE afectado con Windows Server 2019 estaban instalados estos tres paquetes:

- Security Update For Exchange Server Subscription Edition (KB5128895)
- 2026-10 Cumulative Update for .NET Framework 3.5, 4.7.2 and 4.8 for Windows Server 2019 for x64 (KB5128923)
- 2026-10 Cumulative Update for Windows Server 2019 for x64-based Systems (KB5126651)

Microsoft no menciona la actualización de .NET en la adenda. Sin embargo, lectores de Born informan de caídas con la combinación de actualización de Windows y .NET (allí KB5128928 en Windows Server 2022), así como de un servidor con KB5126651 y SU de Exchange, pero sin actualización de .NET, que seguía funcionando después de más de 24 horas. Por tanto, no está claro si la actualización de .NET interviene.

Sobre la SU de Exchange: en foros y comentarios se afirma que la situación mejora al instalar Exchange SU11. Esto no puede confirmarse; las caídas también se produjeron en un servidor con SU11. A la inversa, varios administradores informan de caídas sin una SU de Exchange instalada, lo que coincide con la afirmación de Microsoft. La SU de Exchange corrige seis vulnerabilidades, incluida una ejecución remota de código crítica mediante un correo electrónico preparado; detalles en el [artículo sobre la SU de octubre](/blog/exchange-security-updates-oktober-2026). Retenerla no resuelve el problema.

## Comprobar el estado

Primero, determine en cada servidor Exchange cuáles de las actualizaciones de octubre están instaladas:

```powershell
$kbs = "KB5126646", "KB5126655", "KB5126651", "KB5126868", "KB5128923"
Get-HotFix -Id $kbs -ErrorAction SilentlyContinue |
  Select-Object PSComputerName, HotFixID, InstalledOn
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `$kbs`, `-Id` | Lista de los números KB buscados: las cuatro CU de Windows de la adenda de Microsoft y la actualización de .NET para Windows Server 2019. |
| `-ErrorAction SilentlyContinue` | Suprime el mensaje de error para números KB que no están instalados en este servidor. |
| `Select-Object PSComputerName, HotFixID, InstalledOn` | Muestra servidor, número KB y fecha de instalación; la fecha permite estimar cuándo cabe esperar la próxima caída. |

</details>

`Get-HotFix` no enumera de forma fiable las actualizaciones de .NET en todos los sistemas; en caso de duda, el historial de actualizaciones en Configuración muestra el estado completo. Compruebe la compilación de Exchange en Exchange Management Shell con `Get-ExchangeServer | Format-Table Name, AdminDisplayVersion` y compárela con los [números de compilación de Exchange](/tools/exchange-builds).

A continuación, compruebe el estado de las bases de datos, las copias y las colas en Exchange Management Shell:

```powershell
Get-MailboxDatabaseCopyStatus -Server $env:COMPUTERNAME |
  Format-Table Name, Status, CopyQueueLength, ReplayQueueLength, ContentIndexState
Get-Queue | Where-Object MessageCount -gt 0 |
  Format-Table Identity, DeliveryType, Status, MessageCount, LastError
Get-WinEvent -FilterHashtable @{LogName='Application'; Id=1006; StartTime=(Get-Date).AddDays(-2)} |
  Select-Object TimeCreated, ProviderName, Message
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `Get-MailboxDatabaseCopyStatus -Server $env:COMPUTERNAME` | Estado de todas las copias de bases de datos en el servidor local; sin DAG solo aparece la copia activa. |
| `CopyQueueLength`, `ReplayQueueLength` | Número de archivos de registro que aún no se han copiado o reproducido; los valores crecientes indican que la replicación está detenida. |
| `ContentIndexState` | Estado del índice de búsqueda de la copia. |
| `Get-Queue \| Where-Object MessageCount -gt 0` | Muestra únicamente las colas que contienen correos. |
| `LastError` | Último error de la cola; en este problema, normalmente errores de conexión con la base de datos de buzones. |
| `-FilterHashtable @{LogName='Application'; Id=1006; …}` | Busca en el registro Application el Event ID 1006, tal como indica Microsoft. |
| `StartTime=(Get-Date).AddDays(-2)` | Limita la búsqueda a los dos últimos días, es decir, al periodo desde el Patch Tuesday. |

</details>

Varios componentes de Exchange registran el Event ID 1006; lo decisivo es la entrada cuyo texto contiene el mensaje MdbOffline.

## Procedimiento hasta la corrección

Los siguientes puntos proceden del 16 de octubre de 2026, cuando aún no existía una medida oficial. Desde el 17 de octubre de 2026, la medida del registro indicada en la actualización anterior es prioritaria; los puntos 3 a 6 siguen siendo válidos como alternativa de contingencia.

1. **Servidores Exchange aún no actualizados**: originalmente se recomendaba aplazar las actualizaciones de Windows de octubre. Con la medida del registro, esto ya no es necesario: establezca el valor e instale las actualizaciones, que también corrigen vulnerabilidades de seguridad.

2. **Instalar igualmente la SU de Exchange**: según Microsoft, la SU de Exchange KB5128895 (o los paquetes ESU para 2016/2019) no es el desencadenante y corrige una vulnerabilidad crítica explotable por correo electrónico sin necesidad de una cuenta.

3. **Servidores ya afectados**: un reinicio restablece el funcionamiento durante aproximadamente un día. Planifique reinicios controlados a intervalos claramente inferiores a 22 horas, por ejemplo cada noche, en lugar de esperar a la siguiente caída. Según lectores de Born, reiniciar únicamente los servicios de Exchange también ayuda temporalmente; no está documentado durante cuánto tiempo.

4. **DAG**: reinicie los miembros uno tras otro y traslade antes las bases de datos activas a un servidor recién reiniciado. Después de cada reinicio, compruebe con `Get-MailboxDatabaseCopyStatus` que las copias vuelvan a estar `Healthy` y que las colas se reduzcan antes de reiniciar el siguiente servidor.

5. **Supervisar las colas**: vigile `Get-Queue` y el Event ID 1006, idealmente con una alerta para que una caída no se detecte solo por avisos de usuarios.

6. **Comprobar las copias de seguridad**: un lector de Born informa de un VSS Writer fallido en la copia de seguridad tras la caída. Revise las copias de seguridad de las últimas noches.

Sobre la desinstalación de las actualizaciones de Windows: desde 2021, Microsoft distribuye la actualización acumulativa junto con la Servicing Stack Update como paquete combinado. `wusa /uninstall` no elimina dicho paquete; para ello se necesita `DISM /Online /Remove-Package` con el nombre de paquete de `DISM /Online /Get-Packages`. Desde el 17 de octubre de 2026, Microsoft desaconseja expresamente la desinstalación; con la medida del registro ya no hay motivo para ello.

Microsoft ya incluye el problema como Known Issue en las notas de la versión de las actualizaciones de Windows (estado del 19 de octubre de 2026, sin resolver). Este artículo se actualizará en cuanto haya una corrección.

## Fuentes

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-october-2026-exchange-server-security-updates/4563014): adenda del 15 de octubre de 2026 con las actualizaciones de Windows afectadas, el mensaje de error, el Event ID 1006 y la indicación de que la SU de Exchange no es necesaria para que se produzca el problema.

2.  [Exchange Server SE: Problems with the October updates for Windows Server – Frankys Web](https://www.frankysweb.de/en/exchange-server-se-problems-with-the-october-update-kb5128895/): caída tras aproximadamente 24 horas, bases de datos aún mostradas como montadas en EAC, reinicio como solución temporal, comentarios sobre reinicios de DAG.

3.  [Achtung: Exchange Server SE Okt. 2026-SU führen zu Ausfällen – Borns IT- und Windows-Blog](https://borncity.com/blog/2026/10/15/achtung-exchange-server-okt-2026-su-fuehren-zu-ausfaellen/): informes de lectores sobre el momento, los KB implicados (también .NET), el servicio de replicación, el modo de mantenimiento y errores VSS.

4.  [Oktober-Updates setzen Exchange-Server ausser Betrieb – IT Magazine](https://www.itmagazine.ch/artikel/88263/Oktober-Updates_setzen_Exchange-Server_ausser_Betrieb.html): resumen suizo de la confirmación de Microsoft.

5.  [Exchange servers may become unstable after installation of October 2026 Windows updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/exchange-servers-may-become-unstable-after-installation-of-october-2026-windows-/4565382): entrada del equipo de Exchange del 17 de octubre de 2026 con el valor de registro EpmapSavedContextCap, la recomendación de no desinstalar las actualizaciones de Windows y la nota sobre los efectos dependientes de la carga.

6.  [October 13, 2026: KB5126646 (OS Build 26100.33650) – Microsoft Support](https://support.microsoft.com/en-us/servicing/os/windows-server/2026/10/kb5126646-windows-server-2025-security-update): Known Issue con el límite de 512 llamadas de búsqueda de endpoints, estado sin resolver y otra medida a través de Microsoft Support for business.

7.  [Microsoft analysiert Exchange-Server-Probleme durch Windows-Oktober-2026-Updates – Borns IT- und Windows-Blog](https://borncity.com/blog/2026/10/17/microsoft-analysiert-exchange-server-probleme-durch-windows-oktober-2026-updates/): análisis de un lector de RpcEpMap.dll (EpmapSavedContextCap, Feature-ID), declaración de Nino Bilic y experiencias con 65536 y valores superiores.

8.  [CVE-2026-96300 – OpenCVE](https://app.opencve.io/cve/CVE-2026-96300): vulnerabilidad de denegación de servicio de RPC Endpoint Mapper Service, CWE-770, publicada el 13 de octubre de 2026.

9.  [Exchange-Sicherheitsupdates vom Oktober 2026 – rafaelpfister.ch](/blog/exchange-security-updates-oktober-2026): contenido de la SU de Exchange KB5128895 y los paquetes ESU, vulnerabilidades e indicaciones de instalación.
