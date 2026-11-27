---
title: "Facilitator en Microsoft Teams: qué puede hacer el agente de reuniones y qué no"
navTitle: "Teams Facilitator"
description: "Facilitator es un agente de Copilot que redacta notas compartidas en reuniones programadas de Teams, gestiona la agenda, controla el tiempo y responde preguntas de forma visible para todos. El artículo explica sus funciones, requisitos de licencia y administración, casos de uso útiles y limitaciones relativas a tipos de reunión, idioma y cumplimiento normativo."
date: "2026-11-27"
kategorie: "Microsoft Teams"
timeToRead: "8 min de lectura"
themen:
  - microsoft-teams
  - microsoft-365-exchange
produkte:
  - "teams"
protokolle:
  - "lizenzierung"
  - "powershell"
hauptthema: "microsoft-teams"
slug: "facilitator-en-microsoft-teams-que-puede-hacer-el-agente-de-reuniones-y-que-no"
translationId: "article-33919f700e0a6587"
translationOf: teams-facilitator
url: https://rafaelpfister.ch/es/blog/facilitator-en-microsoft-teams-que-puede-hacer-el-agente-de-reuniones-y-que-no
translationSourceHash: 796c74d7d8547585c4342621dfa84e867cf70a2cbc477de7f8e82cd88cbc2d0c
translationModel: gpt-5.6-terra
translatedAt: 2026-11-27T09:40:58.141Z
translationReview: required
---

Desde que Microsoft está implementando Facilitator en las reuniones de Teams, en muchos tenants aparece una nueva entrada en las opciones de reunión: «Allow Copilot and Facilitator». El nombre no deja claro qué hay detrás. Facilitator es un agente de IA que participa en la reunión y trabaja de forma visible para todos. De ello se derivan tanto sus posibilidades de uso como sus limitaciones.

## Qué es Facilitator

Facilitator es un agente de Microsoft 365 Copilot que asume la función de asistente de moderación en una reunión programada de Teams. Durante la reunión, redacta notas en una página compartida de Loop que todos los participantes internos pueden leer y editar. Lee la agenda de la invitación, muestra un temporizador de agenda en el escenario de la reunión y responde a las @menciones en el chat de la reunión.

La diferencia con las demás funciones de IA de Teams reside en la visibilidad:

| Función | Quién ve el resultado | Momento |
|---|---|---|
| Copilot en reuniones de Teams | solo la persona que pregunta | durante y después de la reunión |
| Recap inteligente | participantes con licencia de Teams Premium o Copilot | después de la reunión |
| Facilitator | todos los participantes internos | durante la reunión; las notas también después |

Una pregunta a Copilot sigue siendo privada. Una pregunta a Facilitator y su respuesta aparecen en el chat de la reunión para todos. Así, Facilitator trabaja de forma conjunta para todos los participantes.

## Resumen para usuarios

Para todas las funciones se aplica lo siguiente: la reunión debe ser una reunión programada de Teams de su propia organización, sin participantes externos; en las opciones de reunión debe estar activado «Permitir Copilot y Facilitator», y al inicio debe haberse seleccionado el idioma hablado. Solo necesita una licencia de Copilot quien inicie Facilitator o le encargue tareas. Todos los demás participantes internos ven los resultados sin necesidad de una licencia propia de Copilot.

| Función | Qué puede hacer con ella | Quién puede usarla | Requisitos adicionales |
|---|---|---|---|
| Iniciar Facilitator | Activar el agente antes o durante la reunión | Personas con licencia de Microsoft 365 Copilot | Loop activado en Teams; aplicación «Facilitator» no bloqueada |
| Notas en directo | Las decisiones, preguntas abiertas y tareas se registran continuamente en la pestaña «Notes»; puede corregirlas y completarlas directamente | Leer y editar: todos los participantes internos | La transcripción está en marcha (se inicia automáticamente); solo se registra lo hablado en el idioma seleccionado, no los mensajes del chat |
| Agenda y marcado de puntos | La lista de puntos de la invitación aparece en el chat y los puntos tratados se marcan como completados | Ver: todos los participantes internos | La agenda está en la invitación o se proporciona mediante @Facilitator en el chat; invitación no cifrada; además de la persona organizadora, hay al menos otra persona en la reunión |
| Temporizador y recordatorios | Cuenta atrás para un punto, aviso a mitad de tiempo, antes del final de la reunión y en caso de excederlo | Solicitar temporizador: licencia de Copilot; ver: todos los participantes internos | ninguno |
| Preguntas sobre la reunión | Preguntar mediante @Facilitator en el chat, por ejemplo «¿Qué decidimos sobre el punto 2?»; la pregunta y la respuesta son visibles para todos | Preguntar: licencia de Copilot; leer respuestas: todos los participantes internos | Preguntas tras el final de la reunión solo con la opción «Durante y después de la reunión» |
| Preguntas con búsqueda web | Investigación breve durante la reunión, por ejemplo sobre una versión de producto | Licencia de Copilot | La búsqueda web para Copilot no está desactivada en el tenant |
| Crear documento de Word | Generar un documento a partir de las notas, por ejemplo como borrador de acta | Licencia de Copilot | Almacenamiento en el OneDrive de la persona solicitante |
| Crear presentación de PowerPoint | Generar una presentación a partir de la reunión | Licencia de Copilot | Vista previa pública: participación en Teams Public Preview |
| Transferir tareas a Planner | Transferir a Planner las tareas registradas con «Accept to sync» | Licencia de Copilot | Vista previa pública: participación en Teams Public Preview |
| Notas en reuniones presenciales (smartphone) | Documentar una conversación presencial con la aplicación de Teams en el smartphone | Licencia de Copilot | Aplicación de Teams en el smartphone |
| Notas en Teams Room | Mostrar chat, notas y temporizador en la pantalla de la sala | Licencia de Copilot solo para la persona organizadora | Teams Rooms on Windows; para reuniones presenciales no programadas, Teams Rooms Pro (vista previa pública) |

## Funciones y estado

Microsoft clasifica las funciones por separado en su documentación de administración como disponibles de forma general y en vista previa pública (a noviembre de 2026):

| Función | Estado |
|---|---|
| Notas en directo en la pestaña «Notes» (página de Loop, editable de forma colaborativa) | disponibilidad general |
| Responder preguntas a partir del contenido de la reunión y mediante búsqueda web | disponibilidad general |
| Temporizador de agenda con avisos a mitad de tiempo y antes del final de la reunión | disponibilidad general |
| Crear un documento de Word a partir de la reunión | disponibilidad general |
| @mención de Facilitator en el chat de la reunión | disponibilidad general |
| Notas en el smartphone para reuniones presenciales | disponibilidad general |
| Teams Rooms: chat, notas y temporizador en la sala | disponibilidad general |
| Registrar tareas y sincronizarlas con Planner | vista previa pública |
| Crear una presentación de PowerPoint a partir de la reunión | vista previa pública |
| Notas para reuniones presenciales no programadas en Teams Rooms | vista previa pública |

Sobre la agenda: si la invitación contiene una lista de puntos, Facilitator la adopta, la publica en el chat y marca los puntos en cuanto se han tratado. El marcado se realiza a intervalos, no de inmediato. Si la reunión se prolonga más allá de la hora prevista de finalización, lo indica. Si se le solicita, también establece una cuenta atrás para un punto concreto.

Sobre las notas: Facilitator resume decisiones, preguntas abiertas y tareas. La base es exclusivamente lo hablado mediante la transcripción. Lo que solo aparece en el chat de la reunión no se incorpora a las notas. Las notas se pueden corregir y completar manualmente en cualquier momento; Facilitator sigue escribiendo en la misma página.

## Requisitos

Para utilizarlo deben cumplirse varias condiciones simultáneamente:

- **Licencia:** Quien añada Facilitator a una reunión o lo active durante ella necesita una licencia de Microsoft 365 Copilot además de una licencia adecuada de Microsoft 365 y Teams. Los participantes sin licencia de Copilot no pueden hacer preguntas a Facilitator, pero ven las notas, las preguntas de otras personas y sus respuestas.
- **Loop:** Las experiencias de Loop en Teams deben estar activadas en el tenant. Sin Loop no hay notas.
- **Transcripción:** Al activar Facilitator, la transcripción se inicia automáticamente. Si la opción de reunión «Allow Copilot and Facilitator» está configurada como «Only during the meeting», Facilitator funciona sin una transcripción guardada; en ese caso no es posible hacer preguntas después de finalizar la reunión.
- **Permiso de aplicación:** La aplicación «Facilitator» está permitida de forma predeterminada, siempre que no se hayan bloqueado todas las aplicaciones en el Teams Admin Center.

En la invitación, active Facilitator en **Options** > **Copilot and other AI** con la configuración «Allow Copilot and Facilitator». Como alternativa, puede activarlo durante la reunión desde el menú de Copilot.

## Facilitator aparece atenuado

En la reunión, Teams muestra tres interruptores en el menú «IA de reunión»: Copilot, Facilitator y Resumen inteligente. Si los tres están atenuados y aparece el aviso «No tiene acceso a IA en esta reunión», la causa está en la licencia o en la propia reunión. Si solo Facilitator está atenuado, la causa está en la aplicación o en Loop.

| Síntoma | Posible causa | Comprobación |
|---|---|---|
| Los tres interruptores en gris | La cuenta no tiene ni una licencia de Microsoft 365 Copilot ni de Teams Premium | Microsoft 365 Admin Center > Usuarios > Licencias. Una licencia asignada recientemente suele surtir efecto en Teams solo tras unas horas o al iniciar sesión de nuevo |
| Los tres interruptores en gris | Reunión instantánea («Reunirse ahora»), reunión iniciada desde el chat, reunión de canal o llamada | Cree la reunión como reunión programada de Teams desde el calendario |
| Los tres interruptores en gris | Reunión de otra organización, participación como persona externa o invitada | Facilitator no está disponible en reuniones externas |
| Los tres interruptores en gris | La opción de reunión «Permitir Copilot y Facilitator» está en «Desactivado», configurada por la persona organizadora, una plantilla de reunión o una etiqueta de confidencialidad | Opciones de reunión > Copilot y otra IA |
| Los tres interruptores en gris | La directiva de reuniones de Teams bloquea Copilot o la transcripción | Teams Admin Center > Reuniones > Directivas de reuniones |
| Solo Facilitator en gris | Aplicación «Facilitator» bloqueada, por ejemplo porque todas las aplicaciones están bloqueadas | Teams Admin Center > Aplicaciones de Teams > Administrar aplicaciones |
| Solo Facilitator en gris o sin notas | Loop para notas de reunión desactivado | `Get-SPOTenant` muestra `IsCollabMeetingNotesFluidEnabled` |

La licencia es la causa más frecuente. Sin una licencia de Microsoft 365 Copilot, el interruptor de Facilitator permanece atenuado en todas las reuniones, independientemente de las opciones de reunión y las directivas.

## Para qué es adecuado Facilitator

Facilitator aporta el mayor beneficio en reuniones en las que, de otro modo, alguien toma acta y por ello no puede participar en el debate. Ejemplos habituales:

- **Reuniones de proyecto y de estado con agenda fija:** El temporizador de agenda y los avisos a mitad de tiempo mantienen el horario; las notas contienen decisiones y puntos abiertos sin necesidad de un acta separada.
- **Talleres con decisiones:** La página compartida de Loop es visible durante la reunión. Los participantes corrigen directamente las formulaciones, en lugar de acordar un acta posteriormente por correo electrónico.
- **Preguntas durante la reunión:** Facilitator responde en el chat, de forma visible para todos, a «¿Qué decidimos sobre el punto 2?» o a una breve búsqueda web sobre una versión de producto.
- **Resumen posterior:** A petición, Facilitator crea un documento de Word a partir de las notas y lo almacena en el OneDrive de la persona solicitante.
- **Reuniones presenciales:** Con la aplicación de Teams en el smartphone o en una Teams Room también se documentan las reuniones in situ.

## Lo que Facilitator no puede hacer

Las limitaciones se describen claramente en la documentación de Microsoft y afectan sobre todo a los tipos de reunión, el idioma y el cumplimiento normativo.

**Tipos de reunión no compatibles.** Facilitator solo funciona en reuniones programadas de Teams. No se admiten reuniones instantáneas («Meet now»), reuniones iniciadas desde un chat, llamadas de Teams, reuniones de canal ni chats individuales y grupales. La documentación no indica compatibilidad con seminarios web ni Town Halls.

**Sin participantes externos.** Las notas generadas por IA no están disponibles en reuniones ni chats con personas externas. Por regla general, los participantes externos no ven las notas. Por tanto, Facilitator no puede utilizarse como herramienta de actas en reuniones con clientes o proveedores.

**Un idioma por reunión.** Al activarlo se selecciona el idioma hablado. Las notas solo se generan para los fragmentos en ese idioma. En una reunión que alterna entre alemán y francés faltarán las partes en francés. Si el idioma seleccionado difiere del hablado, no se generan notas. El alemán suizo no figura como idioma propio en la lista de idiomas compatibles con Copilot.

**Sin grabación.** Facilitator activa la transcripción, pero no la grabación. Quien necesite imagen y sonido debe iniciar la grabación por separado.

**Sin agenda de invitaciones protegidas.** Si la invitación está cifrada con una etiqueta de confidencialidad, Facilitator no puede leer la agenda. Lo mismo ocurre mientras solo esté la persona organizadora en la reunión, ya que no lee su copia de la invitación.

**No sustituye un acta formal.** Las notas son un resumen mediante IA de la transcripción. Para reuniones del consejo de administración, entrevistas de personal u otras reuniones con consecuencias jurídicas, sigue siendo necesaria un acta revisada y aprobada.

## Almacenamiento de datos y cumplimiento normativo

Las notas se almacenan como archivo `.loop` en la carpeta «Meetings» del OneDrive de la persona que inició Facilitator. Microsoft las trata como datos de transcripción. Para las personas responsables de cumplimiento normativo, son relevantes tres limitaciones:

- **eDiscovery:** Las notas de reunión generadas por IA no se registran automáticamente como archivos adjuntos en la nube en Microsoft Purview eDiscovery.
- **Retención:** Las etiquetas de retención no son compatibles con los archivos adjuntos en la nube de las notas generadas por IA.
- **Etiquetas de confidencialidad:** Las notas no heredan la etiqueta de la reunión. Se puede aplicar una etiqueta posteriormente al componente de Loop. Configuraciones como «Prevent copy and paste» y las marcas de agua no se aplican a las respuestas ni a las notas de Facilitator.

Para las organizaciones con obligaciones de retención, esto significa que las notas que deban considerarse documentación empresarial deben transferirse a un repositorio con retención operativa. Microsoft describe los aspectos de Purview en una página independiente (véase Fuentes).

## Controlar y desactivar Facilitator

En el Teams Admin Center encontrará la aplicación en **Teams apps** > **Manage apps** buscando «Facilitator». Allí puede permitirla o bloquearla para todo el tenant. Para grupos de usuarios concretos, controle la aplicación mediante directivas de permisos de aplicaciones o administración centrada en aplicaciones.

Las notas de Loop en las reuniones, en las que se basa Facilitator, dependen de una configuración de tenant de SharePoint. Esta se aplica a todos los usuarios y no puede configurarse por persona:

```powershell
Connect-SPOService -Url "https://contoso-admin.sharepoint.com"
Set-SPOTenant -IsCollabMeetingNotesFluidEnabled $false
```

<details class="options-details">
<summary>Explica las opciones</summary>

| Opción | Efecto |
|---|---|
| `-Url` | Dirección del SharePoint Admin Center del tenant (`<tenant>-admin.sharepoint.com`) |
| `-IsCollabMeetingNotesFluidEnabled` | `$false` desactiva las notas compartidas de reunión y, con ello, las notas de IA de Facilitator para todos los usuarios; `$true` las vuelve a activar |

</details>

Esta configuración también elimina la función de notas para reuniones sin Facilitator. Si solo se desea bloquear el agente, bloquear la aplicación en el Teams Admin Center es la vía más específica. Para reuniones sin una transcripción guardada, configure en la directiva de reuniones la opción de Copilot y Facilitator como «Only during the meeting».

## Fuentes

1.  [Set up Facilitator in Microsoft Teams for collaborative AI-generated notes](https://learn.microsoft.com/en-us/microsoftteams/facilitator-teams): Documentación de administración con requisitos de licencia, estado de las funciones, almacenamiento de datos, limitaciones de cumplimiento normativo y control de aplicaciones (a 18/11/2026).

2.  [Facilitator in Microsoft Teams meetings](https://support.microsoft.com/en-us/teams/copilot/facilitator-in-microsoft-teams-meetings): Guía para usuarios finales sobre opciones de reunión, agenda, temporizador, tipos de reunión compatibles y selección de idioma.

3.  [Facilitator in Microsoft Teams Rooms](https://learn.microsoft.com/en-us/microsoftteams/rooms/facilitator-teams-rooms): Reuniones presenciales en Teams Rooms on Windows, licencia de la persona organizadora, asignación de hablantes en la sala.

4.  [Keep meetings focused and productive with Facilitator in Teams](https://techcommunity.microsoft.com/blog/microsoft365insiderblog/keep-meetings-focused-and-productive-with-facilitator-in-teams/4456029): Blog de Microsoft 365 Insider de octubre de 2025 sobre transcripción, base de las notas y funcionamiento sin transcripción guardada.

5.  [Microsoft Purview: Teams Facilitator](https://learn.microsoft.com/en-us/purview/ai-teams-facilitator): Perspectiva de Purview sobre los datos de Facilitator para retención y eDiscovery.

6.  [Manage Loop components in OneDrive and SharePoint](https://learn.microsoft.com/microsoft-365/loop/loop-components-configuration): Configuraciones de tenant `IsLoopEnabled` y `IsCollabMeetingNotesFluidEnabled` para componentes de Loop y notas de reunión.
