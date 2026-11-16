---
title: "Migración: arquitectura objetivo, coexistencia y cutover controlado"
blatt: "migration"
description: "Migración técnica para administradores de mensajería e infraestructura: arquitectura de origen y de destino, dependencias, mapeo de identidades y datos, desde rehost hasta refactor, coexistencia, sincronización inicial y delta, piloto, cutover, límites de responsabilidad, rollback, aceptación y desmantelamiento."
fakten:
  - label: Punto de partida
    wert: transacción de negocio + arquitectura de origen + estados + propietarios
    href: https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html
  - label: Objetivo
    wert: arquitectura objetivo medible con interfaces, soberanía de datos y límites de confianza y de fallo
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Estrategias
    wert: Retain · Retire · Rehost/Relocate · Repurchase · Replatform · Refactor
    href: https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html
  - label: Patrones
    wert: Cutover · Piloto/oleadas · coexistencia · sustitución gradual
    href: https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html
  - label: Identidad
    wert: representar por separado ID de origen, ID de destino, inicio de sesión, direcciones SMTP, grupos, roles y ciclo de vida
    href: https://datatracker.ietf.org/doc/html/rfc7643
  - label: Ruta de datos
    wert: Seed → sincronización delta → Freeze/Final Delta → conciliación
    href: https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration
  - label: Coexistencia
    wert: libreta de direcciones · enrutamiento de correo · Discovery · Auth · permisos · calendario · soporte
    href: https://learn.microsoft.com/en-us/exchange/exchange-hybrid
  - label: Responsabilidad del correo
    wert: 250 después de End-of-Data significa aceptación por el salto SMTP que responde
    href: https://datatracker.ietf.org/doc/html/rfc5321
  - label: Cutover de DNS
    wert: TTL limita la duración de la caché; comprobar por separado cachés negativas y anulaciones locales
    href: https://datatracker.ietf.org/doc/html/rfc1034
  - label: Límite de rollback
    wert: definir explícitamente antes de la primera operación de escritura exclusiva en el destino
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Aceptación
    wert: integridad · completitud · autorización · transacción · rendimiento · monitorización
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Desmantelamiento
    wert: tratar de forma controlada el enrutamiento, las identidades, las claves, las copias de seguridad, los registros, los contratos y los soportes de datos
    href: https://csrc.nist.gov/pubs/sp/800/88/r1/final
werbung:
  - stargate
  - newsletter
ctaThemen:
  - migration
  - messaging
  - exchange-hybrid
translationSourceHash: bdb2fea86e1f811e78f2f210afd26b082f808325322fb675a8ec446e47b38ac3
translationModel: gpt-5.6-terra
translatedAt: 2026-11-13T03:20:43.634Z
translationReview: automatic
---

# Migración: arquitectura objetivo, coexistencia y cutover controlado

Una migración técnica transfiere procesos de negocio, interfaces y estados desde una arquitectura de origen a una arquitectura de destino definida. El cambio visible de producto es solo una parte. Las identidades, direcciones, permisos, colas, datos, claves, DNS, certificados, clientes, automatizaciones, monitorización, copias de seguridad y procesos de soporte deben seguir cumpliendo las mismas garantías funcionales, o trasladarse deliberadamente a un nuevo contrato.

Para los sistemas de mensajería, la garantía más importante es la responsabilidad controlada sobre cada mensaje. Durante una migración, las plataformas antiguas y nuevas pueden conocer direcciones, aceptar conexiones y modificar datos simultáneamente. Sin una soberanía de datos explícita se producen bucles, libretas de direcciones split-brain, entregas duplicadas, permisos perdidos y una vía de retorno que solo existe en la planificación. Por ello, una migración sólida comienza con la arquitectura y los estados, no con la fecha de conmutación.

La explicación comienza con el estado actual documentado y conduce, a través del diseño objetivo, las dependencias y la transferencia de datos, a la coexistencia. El piloto, el cutover, la vía de retorno y la aceptación solo siguen cuando está claro qué datos y funciones se trasladan.

Una migración no es un proceso de copia, sino el cambio controlado de una arquitectura de origen funcional a una arquitectura de destino verificada. Primero se registran los estados y las dependencias; después se planifican la coexistencia, el movimiento de datos, el cutover y la vía de retorno.

## De la arquitectura de origen a la arquitectura objetivo

Antes de elegir el producto, el servicio existente se documenta como un grafo. Cada nodo tiene una función; cada arista tiene iniciador, destino, protocolo, identidad, clase de datos, comportamiento ante errores y propietario. Una lista CMDB de nombres de servidores no basta si se desconoce qué escáner invoca un salto SMTP, qué servicio expande grupos o qué clave descifra mensajes antiguos.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-migration.svg?v=20260813" title="Interaktive Infografik: Migration von Ausgangs- zu Zielarchitektur mit Strategie, Zustandsklassen, Koexistenz, Seed und Delta, Pilotwellen, Cutover, Rollbackgrenze, Abnahme und Stilllegung" loading="lazy">
  <a href="/images/kb-interaktiv-migration.svg?v=20260813">Abrir directamente el gráfico interactivo</a>.
</iframe>

La documentación de origen registra como mínimo:

| Dominio | Estado actual que se debe registrar | Contrato oculto típico |
|---|---|---|
| Enrutamiento de correo | MX, listeners de relay/submission, conectores, smarthosts, dominios aceptados, IP de origen, política TLS | Un partner incluye una IP antigua en una allowlist o espera un nombre de certificado |
| Identidad | Fuente de directorio, ancla de objeto, inicio de sesión, grupos, roles, cuentas de servicio, aprovisionamiento | La aplicación utiliza la dirección de correo electrónico como ID inmutable |
| Datos | Buzón, archivos, journals, cola, cuarentena, blob, índice, configuración, auditoría | El índice de búsqueda es reconstruible, pero el journal está sujeto a conservación obligatoria |
| Criptografía | Certificados TLS, claves S/MIME/OpenPGP, HSM/KMS, frases de contraseña, recuperación | Un mensaje cifrado antiguo requiere una clave privada histórica |
| Clientes y API | Outlook/IMAP/SMTP/API, agentes, móviles, webhooks, scripts | URL, cifrado, puerto, cabecera o campo API codificados de forma rígida |
| Operación | Monitorización, copia de seguridad, restauración, parches, registros, soporte, licencias | La alarma solo comprueba el estado del proceso del sistema antiguo |
| Organización | Propietario, guardia, aprobación, protección de datos, conservación, contrato | Un equipo externo controla DNS o el firewall con un plazo de preparación largo |

La arquitectura objetivo describe los mismos campos como **contrato objetivo**. Además, se indican los límites de confianza, la soberanía de datos, las áreas de fallo, los objetivos de recuperación, el escalado y el estado final previsto del sistema antiguo. NIST SP 800-34 vincula el desarrollo de sistemas con la planificación de contingencias y exige derivar las estrategias y pruebas de recuperación de los requisitos de negocio. Para las migraciones, de ello se desprende que la arquitectura objetivo y la operación de contingencia se diseñan conjuntamente ([NIST SP 800-34 Rev. 1](https://csrc.nist.gov/pubs/sp/800/34/r1/final)).

## Estrategia de migración: ¿qué se modifica realmente?

Las «7 Rs» documentadas por AWS distinguen Retire, Retain, Rehost, Relocate, Repurchase, Replatform y Refactor/Re-architect. Son una taxonomía de migración a la nube, pero pueden aplicarse como una pregunta precisa a cualquier migración de plataforma: ¿la función se mantiene, se elimina, se desplaza sin cambios, se sustituye por un producto, se adapta a una nueva plataforma o se reconstruye funcionalmente? AWS señala que el refactoring es la estrategia más compleja en grandes migraciones y que una modernización separada puede reducir riesgos ([AWS Prescriptive Guidance – Migration Strategies](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html)).

| Estrategia | ¿Qué permanece igual? | ¿Qué cambia? | Ejemplo de mensajería |
|---|---|---|---|
| Retain | Plataforma y responsabilidad | Solo excepción/plazo documentado | El archivo heredado sigue disponible en modo lectura hasta que termine la retención |
| Retire | Nada productivo | Se eliminan la función y los accesos | Se desconecta un endpoint POP3 sin uso |
| Rehost / Relocate | Aplicación y modelo de datos en gran medida | Ubicación de computación/virtualización | La VM MTA se traslada a otra infraestructura |
| Repurchase | Requisito de negocio | Producto, modelo de datos, operación y contrato | Un gateway appliance se sustituye por un gateway SaaS |
| Replatform | Función de negocio y partes de la arquitectura | Runtime, base de datos o modelo operativo | El servicio de correo utiliza una nueva plataforma de base de datos/contenedores |
| Refactor | Solo la intención de negocio | Interfaces, componentes y estados | Una canalización de correo monolítica se reconstruye orientada a eventos |

La estrategia se elige **por componente y clase de datos**. Un proyecto puede replatformar el enrutamiento de correo, retener los datos de archivo, retirar una herramienta de informes y repurchase la interfaz de usuario. Llamarlo todo genéricamente «Lift and Shift» oculta los riesgos realmente diferentes.

## Cutover, oleadas, coexistencia y sustitución gradual

Cuatro patrones de transición son técnicamente distintos y deben derivarse de las capacidades documentadas del producto:

- **Cutover único:** Todas las rutas de escritura cambian en una ventana. Es adecuado cuando la coexistencia no es técnicamente posible, el estado es pequeño y se acepta la interrupción. La vía de retorno debe estar clara antes de la primera operación de escritura en el destino.
- **Piloto y oleadas:** Primero cambia una población representativa; los lotes posteriores siguen tras la aceptación. Los lotes de migración de Microsoft son un ejemplo concreto de transferencia escalonada de buzones ([Microsoft – Staged Email Migration](https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration)).
- **Coexistencia:** Lo antiguo y lo nuevo comparten temporalmente espacio de nombres, libreta de direcciones, enrutamiento o población de usuarios. Exchange Hybrid documenta, entre otras, el flujo seguro de correo, el espacio de nombres de dominio compartido, la libreta de direcciones y el uso compartido de calendarios como funciones de coexistencia ([Microsoft – Exchange Hybrid Deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)).
- **Sustitución gradual:** Una capa de enrutamiento o integración intercepta funciones y las desplaza gradualmente al destino. Martin Fowler describió este patrón en 2004 como «Strangler Fig Application» para reducir el riesgo de un único cutover de reescritura masiva ([Martin Fowler – Original Strangler Fig Application](https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html)).

Ninguno de estos patrones es universalmente seguro. La coexistencia reduce el alcance del cutover, pero genera estados adicionales, reglas de enrutamiento y casos de soporte. La escritura dual puede aumentar la disponibilidad, pero crea problemas de resolución de conflictos y orden. Un cutover único es más sencillo de entender, pero amplía el radio de impacto simultáneo. Por ello, la decisión especifica la evidencia del producto, la duración permitida, la soberanía de datos y los criterios de salida.

Tras elegir la estrategia, debe quedar definido para cada clase de datos qué sistema puede escribir durante cada fase. Dos fuentes de verdad no coordinadas generan conflictos que una herramienta de copia no puede resolver.

## Clases de estado y soberanía de datos

Antes de cada sincronización se define qué datos son **fuente autoritativa**, **copia replicada**, **proyección derivada** o **solo archivo histórico**:

| Clase de estado | Autoridad durante la coexistencia | Pregunta de conflicto/recuperación |
|---|---|---|
| Datos maestros de identidad | Exactamente un directorio líder o sistema HR | ¿Qué ancla de objeto sobrevive a un cambio de nombre y de tenant? |
| Direcciones SMTP y objetos de enrutamiento | Una fuente de direcciones aprobada | ¿Cómo se evitan duplicados y reglas catch-all/alias? |
| Contenido de buzón | Origen o destino definido de forma inequívoca por cada buzón migrado | ¿Puede seguir escribiéndose en el buzón de origen después del cutover? |
| Cola | El MTA que acepta asume la responsabilidad | ¿Cómo se vacían las colas antiguas sin reinyectarlas dos veces? |
| Política y configuración | Objetivo declarativo o master designado | ¿Qué reglas son semánticamente equivalentes y no solo copiadas sintácticamente? |
| Claves y certificados | KMS/HSM o propietario de clave definido | ¿Qué claves privadas históricas deben seguir siendo legibles? |
| Auditoría, journal y retención | Fuente de conservación inmutable | ¿Se pueden demostrar el tiempo, la integridad, el Legal Hold y la exportación? |
| Índice/caché | Proyección reconstruible | ¿Cuánto tarda la reconstrucción y cuándo se considera completa la búsqueda? |

«Sincronizado bidireccionalmente» no es un modelo de datos sin una regla de conflicto. Para cada campo se documentan dirección, frecuencia, transformación, almacenamiento de errores y comportamiento ante cambios simultáneos. Si la herramienta no puede combinar cambios sin pérdida, una parte pasa a modo de solo lectura durante la fase correspondiente o solo se modifica mediante una ruta centralizada de escritura.

## Identidad, direcciones y permisos

El nombre de usuario, la dirección de correo electrónico y el ID técnico del objeto son cosas distintas. RFC 7643 define para SCIM, entre otras cosas, `id` como un identificador asignado por el Service Provider, inmutable y único dentro del tenant; `externalId` puede contener una correlación gestionada por el cliente. Esto ilustra una regla central de migración: una dirección modificable no es un ancla de objeto segura ([RFC 7643](https://datatracker.ietf.org/doc/html/rfc7643)).

Un mapeo de identidad contiene como mínimo ID de origen, ID de destino, `externalId` u otra ancla de correlación, inicio de sesión/UPN, dirección SMTP principal, alias, grupos, roles, atributos de responsable/organización, cuentas de servicio, método de autenticación y estado de aprovisionamiento. RFC 7644 describe operaciones SCIM de creación, modificación, búsqueda y procesamiento masivo; no obstante, la plataforma de destino concreta puede tener campos obligatorios, reglas de unicidad y límites de tasa diferentes ([RFC 7644](https://datatracker.ietf.org/doc/html/rfc7644)).

Los permisos se migran como relaciones, no como listas de texto. La delegación, Send As, Send on Behalf, ACL de carpetas, Shared Mailbox, reserva de recursos, Application Access y el rol de administrador pueden referirse a distintos tipos de objeto. Cada relación necesita un ancla de destino resuelta; las referencias huérfanas o cíclicas se identifican antes de la oleada. Microsoft documenta, por ejemplo, que la migración de buzones entre tenants requiere objetos de destino `MailUser` y determinados atributos, y que solo transfiere contenidos/permisos definidos ([Microsoft – Cross-tenant Mailbox Migration](https://learn.microsoft.com/en-us/microsoft-365/migration/cross-tenant-mailbox-migration?view=o365-worldwide)).

## Ruta de datos: Seed, Delta, Freeze y conciliación

Los estados grandes normalmente se precargan antes del cutover. El proceso sólido es:

1. **Seed:** inventario inicial completo con recuento de objetos, volumen en bytes, lista de errores y rango temporal.
2. **Sincronización delta:** transferir repetidamente los cambios desde el Seed; observar el retraso y la tasa de error.
3. **Freeze o redirección de escritura:** detener rutas de escritura de origen definidas con precisión o dirigirlas al destino.
4. **Delta final:** incorporar los cambios restantes y asegurar una marca de agua reproducible.
5. **Cutover:** cambiar el enrutamiento, Discovery y el acceso de los usuarios al destino.
6. **Conciliación:** comparar cantidad, tamaño, atributos clave, permisos, muestras y transacciones de negocio.

La migración escalonada de correo de Microsoft documenta lotes, estado de sincronización y el requisito de esperar al menos otra sincronización tras el cambio de enrutamiento antes de eliminar un lote. Es evidencia específica del producto para fases de Seed/Delta/finalización, no una instrucción universal para otros sistemas ([Microsoft – Staged Email Migration](https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration)).

Las sumas de comprobación solo demuestran igualdad de bytes con una serialización idéntica. La exportación/importación puede normalizar mensajes, transformar metadatos o estructurar los contenedores de otro modo. Por ello, la aceptación combina sumas técnicas con invariantes de negocio: número de objetos por clase, marca de tiempo más antigua/más reciente, tamaño total, objetos ausentes/mayores, relaciones de permisos, muestra aleatoria y casos deliberadamente difíciles como mensajes MIME grandes, recurrencias de calendario, Unicode, cifrado y Legal Hold.

Una vez definidas la identidad y la soberanía de datos, el tráfico de mensajes en curso puede conducirse entre el sistema antiguo y el de destino. La coexistencia es una arquitectura de producción temporal con sus propios casos de enrutamiento y error.

## Arquitectura de coexistencia para mensajería

Un dominio SMTP compartido en dos plataformas requiere una decisión inequívoca de destinatario y enrutamiento. Microsoft describe para la operación híbrida un espacio de nombres de dominio compartido, flujo seguro de correo, libreta de direcciones compartida y funciones de calendario; para la coexistencia con terceros se destacan explícitamente las decisiones de identidad, sincronización y autenticación ([Microsoft – Exchange Hybrid Deployments](https://learn.microsoft.com/en-us/exchange/exchange-hybrid), [Microsoft – Third-party Email Coexistence](https://learn.microsoft.com/en-us/exchange/mailbox-migration/plan-to-coexist-with-third-party-messaging-and-aad-ds)).

La matriz local de coexistencia responde a:

- ¿Qué plataforma conoce a cada destinatario y con qué tipo?
- ¿Quién es responsable de los destinatarios desconocidos y cómo se evitan los bucles?
- ¿Qué salto firma DKIM, aplica DLP/archivo/disclaimer o cifra?
- ¿Cómo reconocen lo antiguo y lo nuevo al otro como salto interno, autenticado o externo?
- ¿Dónde residen Free/Busy, la delegación, la expansión de grupos y Autodiscover/Service Discovery?
- ¿Qué plataforma escribe qué atributos de identidad y dirección?
- ¿Cómo se correlacionan bounces, cuarentena, journals y Message Trace entre ambas partes?
- ¿Qué punto de soporte tiene una visión de extremo a extremo?

Cada transición se hace observable mediante respuesta [SMTP](/kb/smtp)-, ID de cola e ID de trace/mensaje. RFC 5321 establece que el servidor receptor asume la responsabilidad tras completar DATA correctamente. Un rollback tras una respuesta `250` no debe olvidar este mensaje aceptado ni reinyectarlo sin control ([RFC 5321](https://datatracker.ietf.org/doc/html/rfc5321)).

## DNS, certificados y contratos externos

DNS es un mecanismo de selección distribuido, no un interruptor global. RFC 1034 describe la caché como un componente fundamental y deja a la fuente de datos el compromiso de TTL entre actualidad y esfuerzo. Por ello, antes de una transición se reducen oportunamente los TTL relevantes; después del cutover, las rutas antiguas permanecen controladas al menos mientras los resolvers, las cachés negativas, los archivos hosts locales, los proxies y los gateways externos puedan seguir utilizando datos antiguos ([RFC 1034](https://datatracker.ietf.org/doc/html/rfc1034)).

Deben comprobarse MX, A/AAAA, CNAME, SRV y TXT, así como las vistas internas/externas. SPF debe autorizar las nuevas IP de envío sin eliminar demasiado pronto las fuentes antiguas; el selector DKIM y la alineación DMARC se verifican con mensajes reales. Los certificados requieren nombres, cadena, confianza, clave privada, Extended Key Usage y capacidad de exportación/HSM. Las claves históricas de descifrado S/MIME/OpenPGP siguen su propio ciclo de vida. NIST SP 800-57 aborda los estados, la protección y el archivado de claves; un cambio de plataforma no elimina estos requisitos ([NIST SP 800-57 Part 1 Rev. 5](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)).

Los contratos externos incluyen allowlisting de partners, credenciales de smarthost, URI de redirección de API, webhooks, fuentes SIEM, agentes de copia de seguridad, sondas de monitorización, firewalls, NAT, DNS inverso y accesos de soporte. Cada contrato tiene un propietario técnico y una prueba antes del cutover.

## Piloto, oleadas y puertas de decisión

Un piloto es representativo, no solo seleccionado por buena disposición. Incluye buzones grandes y pequeños, delegaciones, Shared/Resource Mailboxes, clientes móviles y antiguos compatibles, partners externos, cifrado, archivos, Power Users, distintas ubicaciones y al menos un caso negativo controlado. Al mismo tiempo, la población piloto no debe contener ninguna función exclusiva irreversible.

Cada oleada tiene puertas:

- **Ready:** Mapeo completo, objetos de destino presentes, Seed dentro del objetivo de retraso, cliente/red/política listos, restauración y soporte disponibles.
- **Go:** Última comprobación delta y de integridad correcta, sin ninguna clase de bloqueador abierta, dependencias y ventana de tiempo confirmadas.
- **Hold:** Tasa de error, retraso, cola, replicación o carga de soporte fuera del umbral; no se inician más objetos.
- **Abort:** Integridad de datos, seguridad, pérdida de correo no controlable u otros criterios rojos definidos previamente.
- **Accept:** Se cumplen la aceptación funcional y el periodo de observación posterior; el objeto pertenece ahora inequívocamente al destino.

Las oleadas se segmentan de modo que una interrupción quede limitada: por ejemplo, por ubicación, tenant, dominio, base de datos, tipo de cliente, grupo funcional o dependencia. Los conjuntos aleatorios pueden ser representativos, pero son operativamente difíciles de revertir. Cada oleada recibe una lista inmutable de objetos y un registro de resultados.

El piloto y las oleadas aportan la evidencia para el cutover propiamente dicho. El retorno solo sigue siendo realista mientras los cambios de estado puedan devolverse íntegramente al sistema antiguo.

## Cutover y límite de rollback

El runbook de cutover no contiene solo pasos, sino **condiciones previas, propietario, hallazgo esperado, duración máxima, criterio de cancelación y vía de retorno**. Una posible secuencia es:

1. Congelar cambios, abrir el canal de incidentes/comunicación y guardar las métricas de referencia.
2. Comprobar el Delta final y los errores de mapeo; bloquear las rutas de escritura de origen conforme al modelo.
3. Volver a probar los listeners de destino, certificados, identidad, política y dependencias downstream.
4. Cambiar los controles internos de enrutamiento/Discovery y, después, las rutas externas DNS/de partners.
5. Ejecutar transacciones controladas de inbound, outbound, Submission, Access, Policy y cifrado.
6. Observar cola, códigos SMTP, autenticación, replicación, índice, errores de cliente y monitorización.
7. Decidir Go/Hold/Abort y documentar el momento.

El **Rollback Point** se sitúa antes del momento en que el destino asume la responsabilidad exclusiva de escritura que no puede revertirse sin pérdida. Después, un cambio ciego a la fuente no es un rollback, sino una segunda migración. Los nuevos mensajes de destino, cambios de calendario, permisos, claves y estados de cola deben resincronizarse, exportarse o conservarse mediante roll-forward. El [modelo de Backup y DR](/kb/backup-dr) define el punto de consistencia, RPO/RTO y el orden de recuperación.

## Herramientas de administración para evidencias antes/después

Los ejemplos comprueban contratos individuales. Se ejecutan desde el origen realmente afectado y con datos de prueba aprobados. Las credenciales, el contenido personal y las claves privadas no deben incluirse sin protección en tickets ni en el historial de shell.

### Comparar DNS antes y después del cutover

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-DNS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName example.ch -Type MX -DnsOnly -Server 192.0.2.53
Resolve-DnsName autodiscover.example.ch -Type CNAME -DnsOnly -Server 192.0.2.53
Resolve-DnsName example.ch -Type TXT -DnsOnly -Server 192.0.2.53</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">dig @192.0.2.53 +noall +answer example.ch MX
dig @192.0.2.53 +noall +answer autodiscover.example.ch CNAME
dig @192.0.2.53 +noall +answer example.ch TXT</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname) y [`dig`](https://bind9.readthedocs.io/en/latest/manpages.html#dig-dns-lookup-utility) hacen explícitos el resolver y el tipo de registro. Los resolvers internos y externos se comprueban por separado; una respuesta correcta de un resolver público no demuestra que el servicio de producción use la misma vista.

### Alcanzar nuevos endpoints desde el origen real

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-TCP-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection new-mx.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection new-api.example.ch -Port 443 -InformationLevel Detailed</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 new-mx.example.ch 25
nc -vz -w 5 new-api.example.ch 443</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) solo demuestran la accesibilidad DNS/TCP desde el origen elegido. Firewall, NAT, IP de origen, TLS, autenticación y protocolo funcional se comprueban por separado después.

### Comprobar la identidad TLS y la respuesta del protocolo

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-TLS-Prüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">openssl.exe s_client -connect new-mx.example.ch:25 -starttls smtp -servername new-mx.example.ch -showcerts
curl.exe --verbose --connect-timeout 5 https://new-api.example.ch/health</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">openssl s_client -connect new-mx.example.ch:25 -starttls smtp -servername new-mx.example.ch -showcerts
curl --verbose --connect-timeout 5 https://new-api.example.ch/health</code></pre>
  </div>
</div>

[`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) y [`curl`](https://curl.se/docs/manpage.html) comprueban TLS y un diálogo controlado. Ambos utilizan su propia configuración de confianza/proxy/build; además, el cliente de producción debe probarse con su política real.

### Exportar y comparar el mapeo de identidades

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-Identitätsprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ADUser -Filter * -Properties ObjectGUID,UserPrincipalName,mail,proxyAddresses,Enabled |
  Select-Object ObjectGUID,SamAccountName,UserPrincipalName,mail,proxyAddresses,Enabled |
  Export-Csv .\\identity-source.csv -NoTypeInformation -Encoding UTF8</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ldapsearch -LLL -x -H ldaps://directory.example.ch \
  -b 'dc=example,dc=ch' '(objectClass=person)' \
  entryUUID uid userPrincipalName mail proxyAddresses</code></pre>
  </div>
</div>

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) y [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch) proporcionan atributos de identidad de los directorios respectivos. La exportación y la consulta deben tener en cuenta el ámbito, la paginación, objetos eliminados/desactivados, atributos multivalor y permisos de acceso. La conciliación de destino utiliza el ancla de objeto acordada, no solo el nombre visible.

### Verificar artefactos de exportación byte a byte

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-Hashprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-FileHash .\\export\\mailbox-001.pst -Algorithm SHA256
Get-FileHash .\\export\\mapping.csv -Algorithm SHA256</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sha256sum ./export/mailbox-001.mbox
sha256sum ./export/mapping.csv</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) y [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) detectan cambios en archivos idénticos. No demuestran que los buzones de origen y destino sean semánticamente iguales; para ello se necesitan recuentos de objetos, atributos, listas de errores y muestras funcionales.

### Conservar registros en la ventana temporal de cutover

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Migrations-Logprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$since = [datetime]'2026-08-08T20:00:00Z'
Get-WinEvent -FilterHashtable @{LogName='System'; StartTime=$since} |
  Select-Object TimeCreated,Id,ProviderName,LevelDisplayName,Message</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">journalctl --since '2026-08-08 20:00:00 UTC' \
  --output short-iso-precise --no-pager</code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent) y [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) proporcionan eventos del sistema con referencia temporal. Se añaden registros de producto, lote de migración, SMTP, directorio, base de datos y API con ID de cola/objeto/trace. Antes de reiniciar o rotar, se conservan las evidencias volátiles.

## Aceptación y seguimiento posterior

La aceptación no solo comprueba que «el usuario puede iniciar sesión», sino seis dimensiones:

1. **Completitud:** presentes los objetos, rangos temporales, carpetas, atributos, reglas, claves y permisos esperados.
2. **Integridad:** sin modificación no permitida, duplicado, relación perdida ni soberanía de datos incoherente.
3. **Permisos:** usuarios, servicios y administradores pueden ejecutar exactamente las operaciones previstas, incluidas las pruebas negativas.
4. **Transacción:** inbound/outbound, Submission/Access, enrutamiento, Policy, cifrado, cuarentena, archivo y API funcionan de extremo a extremo.
5. **Operación:** rendimiento, cola, replicación, índice, copia de seguridad, restauración, parche, soporte y capacidad cumplen los valores objetivo.
6. **Observabilidad:** registros, métricas, traces, Message Tracking, alertas y runbooks ven la ruta de destino.

La fase de seguimiento posterior abarca ciclos relevantes de TTL, caché, reintento, delta, retención y carga de negocio. Las plataformas antiguas permanecen activas solo en la función previamente definida: por ejemplo, vaciar una cola o permitir lectura histórica, no seguir aceptando nuevos datos inadvertidamente. Los forwarders temporales y los dominios de coexistencia reciben una fecha de caducidad y monitorización.

El desmantelamiento comienza solo después de la aceptación funcional y el seguimiento posterior. Los registros, las claves, las entradas DNS, las cuentas de servicio y las integraciones ocultas deben eliminarse o archivarse de forma controlada.

## Desmantelamiento sin dependencias perdidas

El decommissioning es un cambio de estado controlado independiente. Antes de apagar, se comprueban los últimos accesos, colas restantes, sesiones activas, jobs, accesos DNS/firewall, cuentas de servicio, copias de seguridad, obligaciones de restauración, Legal Holds, licencias y contratos de soporte. Después se eliminan el enrutamiento y Discovery, se bloquean los accesos, se revocan o archivan secrets y certificados, se adapta la monitorización y se actualiza la CMDB.

Los soportes de datos y recursos cloud se eliminan o se hacen criptográficamente inutilizables conforme a la clase de datos y el contrato. NIST SP 800-88 describe Clear, Purge y Destroy como métodos de sanitización y exige una decisión adecuada al soporte y al riesgo ([NIST SP 800-88 Rev. 1](https://csrc.nist.gov/pubs/sp/800/88/r1/final)). Las copias de seguridad y las claves pueden durar más que la instancia del producto; su conservación, legibilidad y destrucción definitiva siguen documentadas.

## Evolución técnica de las migraciones

Las primeras migraciones de infraestructura solían consistir en una sustitución única de hardware o software con una gran ventana de conmutación. La virtualización y las copias de imágenes facilitaron Rehost y Relocate; los protocolos estandarizados de directorio, API y aprovisionamiento permitieron una coexistencia más precisa. Las plataformas cloud y SaaS convirtieron los límites de tenant, identidad y contrato en objetos de migración tan importantes como el servidor y la base de datos. La taxonomía de las 7 R describe estas distintas profundidades de intervención; el patrón Strangler Fig describe la sustitución gradual de funciones ([AWS – Migration Strategies](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html), [Martin Fowler – Original Strangler Fig Application](https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html)).

El progreso técnico no elimina el cutover. Lo desplaza a unidades más pequeñas: identidades individuales, dominios, API, clases de datos o decisiones de enrutamiento. Esto reduce el radio de impacto simultáneo, mientras que la coexistencia y la soberanía de datos se vuelven más exigentes. Por ello, la arquitectura moderna de migración no optimiza solo la velocidad de copia, sino la **claridad de la responsabilidad, la observabilidad de cada transición y una vía real de retorno o avance**.

## Fuentes

- [NIST – SP 800-34 Rev. 1, planificación de contingencias](https://csrc.nist.gov/pubs/sp/800/34/r1/final)
- [AWS Prescriptive Guidance – estrategias de migración](https://docs.aws.amazon.com/prescriptive-guidance/latest/large-migration-guide/migration-strategies.html)
- [Microsoft – migración escalonada de correo electrónico](https://learn.microsoft.com/en-us/exchange/mailbox-migration/perform-a-staged-migration/perform-a-staged-migration)
- [Microsoft – implementaciones híbridas de Exchange](https://learn.microsoft.com/en-us/exchange/exchange-hybrid)
- [Martin Fowler – aplicación Strangler Fig original](https://www.martinfowler.com/bliki/OriginalStranglerFigApplication.html)
- [IETF RFC 7643 – esquema central SCIM](https://datatracker.ietf.org/doc/html/rfc7643)
- [IETF RFC 7644 – protocolo SCIM](https://datatracker.ietf.org/doc/html/rfc7644)
- [Microsoft – migración de buzones entre tenants](https://learn.microsoft.com/en-us/microsoft-365/migration/cross-tenant-mailbox-migration?view=o365-worldwide)
- [Microsoft – coexistencia de correo electrónico con terceros](https://learn.microsoft.com/en-us/exchange/mailbox-migration/plan-to-coexist-with-third-party-messaging-and-aad-ds)
- [IETF RFC 5321 – protocolo simple de transferencia de correo](https://datatracker.ietf.org/doc/html/rfc5321)
- [IETF RFC 1034 – nombres de dominio: conceptos e instalaciones](https://datatracker.ietf.org/doc/html/rfc1034)
- [NIST – SP 800-57 Parte 1 Rev. 5, gestión de claves](https://csrc.nist.gov/pubs/sp/800/57/pt1/r5/final)
- [Microsoft Learn – Resolve-DnsName](https://learn.microsoft.com/powershell/module/dnsclient/resolve-dnsname)
- [BIND 9 – dig](https://bind9.readthedocs.io/en/latest/manpages.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [curl – página de manual de la línea de comandos](https://curl.se/docs/manpage.html)
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser)
- [OpenLDAP – ldapsearch](https://www.openldap.org/software/man.cgi?query=ldapsearch)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – utilidades SHA-2](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [NIST – SP 800-88 Rev. 1, sanitización de soportes](https://csrc.nist.gov/pubs/sp/800/88/r1/final)
