---
title: "Caídas de Exchange tras las actualizaciones de Windows Server de octubre de 2026: MapiExceptionMdbOffline (ec=1142)"
navTitle: "Caída de Exchange 10/2026"
description: "Tras las actualizaciones de Windows Server del 13 de octubre de 2026, los servidores Exchange notifican MapiExceptionMdbOffline (ec=1142) después de unas 24 horas: OWA falla, las colas crecen y las copias de DAG dejan de replicarse. Microsoft confirma la relación con las actualizaciones de Windows, pero aún no hay corrección. Síntomas, KB afectados, comandos de comprobación y procedimiento."
date: "2026-10-16"
kategorie: "Exchange OnPrem / Hybrid"
timeToRead: "7 min de lectura"
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
aiPrompt: |
  Du bist mein Assistent für Exchange Server On-Premises. Seit den Windows-Server-Updates vom Oktober 2026 (KB5126646, KB5126655, KB5126651, KB5126868) melden Exchange-Server MapiExceptionMdbOffline (hr=0x80004005, ec=1142), Event ID 1006, wachsende Warteschlangen und nicht replizierende DAG-Kopien. Hilf mir Schritt für Schritt: ermitteln, welche Oktober-Updates (Windows, .NET Framework, Exchange SU) auf jedem Exchange-Server installiert sind, den Zustand von Datenbanken, Kopien und Warteschlangen prüfen, den Betrieb mit kontrollierten Neustarts überbrücken und die Windows-Updates auf noch nicht betroffenen Servern zurückstellen. Frage zuerst nach Exchange-Version, Windows-Server-Version, Anzahl Server, ob eine DAG im Einsatz ist und wie Updates verteilt werden (WSUS, Intune, manuell).
translationOf: exchange-windows-updates-oktober-2026
url: https://rafaelpfister.ch/es/blog/caidas-de-exchange-tras-las-actualizaciones-de-windows-server-de-octubre-de-2026
translationSourceHash: 562adaae54d08dd594a344f91f79a05160af956ab320978cef34097a075cab52
translationModel: gpt-5.6-terra
translatedAt: 2026-10-16T08:19:06.833Z
translationReview: automatic
---

# Caídas de Exchange tras las actualizaciones de Windows Server de octubre de 2026: MapiExceptionMdbOffline (ec=1142)

Desde el Patch Tuesday del 13 de octubre de 2026, los servidores Exchange fallan tras unas 24 horas de funcionamiento: los clientes pierden la conexión con el buzón, OWA muestra `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`, los correos permanecen en las colas y las copias de bases de datos en DAG dejan de replicarse. Microsoft añadió una nota al anuncio de las SU de Exchange de octubre el 15 de octubre de 2026: el desencadenante son las actualizaciones de octubre para Windows Server; no es necesario instalar la SU de Exchange para que se produzca el problema. Aún no hay una corrección ni una solución alternativa oficial (a fecha de 16 de octubre de 2026).

## Síntomas

Los síntomas comunicados hasta ahora:

- OWA falla con `MapiExceptionMdbOffline: Unable to make connection to the server. (hr=0x80004005, ec=1142)`; el registro de aplicaciones contiene el evento con ID 1006.
- Los clientes Outlook pierden la conexión con el buzón.
- Los correos no se entregan ni interna ni externamente, y las colas de transporte crecen.
- En DAG, las copias de bases de datos dejan de replicarse; según informes de lectores en Born, las conmutaciones por error al pasar al modo de mantenimiento fallan.
- El Centro de administración de Exchange sigue mostrando las bases de datos como montadas (Frankys Web).

El error no aparece inmediatamente después de la instalación, sino, según los informes, tras unas 22 a 30 horas de funcionamiento. Reiniciar el servidor restablece el servicio, pero la caída se repite aproximadamente al cabo del mismo tiempo. En comentarios en Born se observa que el servicio de replicación de buzones se detiene durante o después de las tareas de mantenimiento de las bases de datos y que después empiezan los errores MdbOffline; esto no está confirmado.

## Actualizaciones afectadas

En la adenda al anuncio de Exchange, Microsoft menciona estas actualizaciones acumulativas de Windows:

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

Microsoft no menciona la actualización de .NET en la adenda. Sin embargo, lectores de Born informan de caídas con la combinación de actualización de Windows y .NET (allí KB5128928 en Windows Server 2022), así como de un servidor con KB5126651 y SU de Exchange, pero sin actualización de .NET, que seguía funcionando después de más de 24 horas. Por tanto, no está claro si la actualización de .NET está implicada.

Sobre la SU de Exchange: en foros y comentarios se dice que la situación mejora al instalar Exchange SU11. Esto no se puede confirmar; las caídas también se produjeron en un servidor con SU11. A la inversa, varios administradores informan de caídas sin ninguna SU de Exchange instalada, lo que concuerda con la declaración de Microsoft. La SU de Exchange corrige seis vulnerabilidades, incluida una ejecución remota de código crítica mediante un correo electrónico manipulado; detalles en el [artículo sobre la SU de octubre](/blog/exchange-security-updates-oktober-2026). Retenerla no resuelve el problema.

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
| `Select-Object PSComputerName, HotFixID, InstalledOn` | Muestra el servidor, el número KB y la fecha de instalación; a partir de la fecha se puede estimar cuándo cabe esperar la próxima caída. |

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
| `CopyQueueLength`, `ReplayQueueLength` | Número de archivos de registro aún no copiados o aún no reproducidos; los valores crecientes indican que la replicación está detenida. |
| `ContentIndexState` | Estado del índice de búsqueda de la copia. |
| `Get-Queue \| Where-Object MessageCount -gt 0` | Muestra solo las colas que contienen correos. |
| `LastError` | Último error de la cola; en este problema, normalmente errores de conexión con la base de datos de buzones. |
| `-FilterHashtable @{LogName='Application'; Id=1006; …}` | Busca en el registro de aplicaciones el evento con ID 1006, tal como indica Microsoft. |
| `StartTime=(Get-Date).AddDays(-2)` | Limita la búsqueda a los últimos dos días, es decir, al periodo desde el Patch Tuesday. |

</details>

Varias componentes de Exchange registran el evento con ID 1006; el factor decisivo es la entrada con el mensaje MdbOffline en el texto.

## Procedimiento hasta que haya una corrección

Según sus propias declaraciones, Microsoft está investigando el problema junto con el equipo de Windows y hasta ahora no ha indicado ni una solución alternativa ni una reversión. Para el periodo intermedio:

1. **Servidores Exchange aún no actualizados**: Aplace las actualizaciones de Windows de octubre (y la actualización de .NET del mismo día) en los servidores Exchange, en WSUS, Intune o su propia gestión de actualizaciones. Es una decisión de equilibrio: las actualizaciones de Windows también corrigen vulnerabilidades de seguridad. Por ello, los servidores Exchange accesibles desde Internet deben vigilarse especialmente de cerca.

2. **Instale igualmente la SU de Exchange**: Según Microsoft, la SU de Exchange KB5128895 (o los paquetes ESU para 2016/2019) no es el desencadenante y corrige una vulnerabilidad crítica explotable por correo electrónico sin necesidad de una cuenta.

3. **Servidores ya afectados**: Un reinicio restablece el servicio durante aproximadamente un día. Planifique reinicios controlados con un intervalo claramente inferior a 22 horas, por ejemplo cada noche, en lugar de esperar a la siguiente caída. Según lectores de Born, reiniciar solo los servicios de Exchange también ayuda temporalmente; no está documentado durante cuánto tiempo.

4. **DAG**: Reinicie los miembros uno tras otro y traslade previamente las bases de datos activas a un servidor recién reiniciado. Después de cada reinicio, compruebe con `Get-MailboxDatabaseCopyStatus` que las copias vuelvan a estar `Healthy` y que las colas se reduzcan antes de reiniciar el siguiente servidor.

5. **Supervisar las colas**: Vigile `Get-Queue` y el evento con ID 1006, idealmente con una alerta, para que una caída no se detecte solo mediante avisos de usuarios.

6. **Comprobar las copias de seguridad**: Un lector de Born informa de un escritor VSS fallido en la copia de seguridad después de la caída. Revise las copias de seguridad de las últimas noches.

Para desinstalar las actualizaciones de Windows: desde 2021, Microsoft distribuye la actualización acumulativa junto con la actualización de pila de mantenimiento como un paquete combinado. `wusa /uninstall` no elimina ese paquete; para ello se necesita `DISM /Online /Remove-Package` con el nombre del paquete de `DISM /Online /Get-Packages`. Microsoft no ha aprobado la desinstalación como solución alternativa y hasta ahora no hay informes de una reversión satisfactoria seguida de un funcionamiento estable. En todo caso, pruébelo primero en un único servidor.

Según Frankys Web, Microsoft todavía no clasifica el problema como Known Issue en las actualizaciones de Windows (a fecha de 16 de octubre de 2026). Siga el anuncio de Microsoft; allí Microsoft añade las actualizaciones al final bajo «Updates to this blog post». Este artículo se actualizará en cuanto haya una corrección o una solución alternativa oficial.

## Fuentes

1.  [Released: October 2026 Exchange Server Security Updates – Microsoft Community Hub](https://techcommunity.microsoft.com/blog/exchange/released-october-2026-exchange-server-security-updates/4563014): Adenda del 15 de octubre de 2026 con las actualizaciones de Windows afectadas, el mensaje de error, el evento con ID 1006 y la indicación de que la SU de Exchange no es necesaria para que se produzca el problema.

2.  [Exchange Server SE: Problems with the October updates for Windows Server – Frankys Web](https://www.frankysweb.de/en/exchange-server-se-problems-with-the-october-update-kb5128895/): Caída después de unas 24 horas, bases de datos aún mostradas como montadas en el EAC, reinicio como solución temporal y comentarios sobre reinicios de DAG.

3.  [Achtung: Exchange Server SE Okt. 2026-SU führen zu Ausfällen – Borns IT- und Windows-Blog](https://borncity.com/blog/2026/10/15/achtung-exchange-server-okt-2026-su-fuehren-zu-ausfaellen/): Informes de lectores sobre el momento, los KB implicados (también .NET), el servicio de replicación, el modo de mantenimiento y errores VSS.

4.  [Oktober-Updates setzen Exchange-Server ausser Betrieb – IT Magazine](https://www.itmagazine.ch/artikel/88263/Oktober-Updates_setzen_Exchange-Server_ausser_Betrieb.html): Resumen suizo de la confirmación de Microsoft.

5.  [Exchange-Sicherheitsupdates vom Oktober 2026 – rafaelpfister.ch](/blog/exchange-security-updates-oktober-2026): Contenido de la SU de Exchange KB5128895 y los paquetes ESU, vulnerabilidades e indicaciones de instalación.
