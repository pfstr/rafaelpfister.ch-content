---
title: "Licencias: derechos de uso, métricas y puntos técnicos de recuento"
blatt: "lizenzierung"
description: "Licencias técnicas para administradores de infraestructura y mensajería: derechos, métricas de usuarios, dispositivos, instancias, núcleos y capacidad, ediciones, suscripciones, activación, servidores de licencias y API en la nube, fuentes de identidad, HA/DR, límites técnicos de aplicación, datos de auditoría, código abierto y verificación."
fakten:
  - label: Determinante
    wert: Contrato, Product Terms y pedido; la indicación técnica es una prueba, no el contrato legal
    href: https://www.iso.org/standard/52293.html
  - label: Tres cantidades
    wert: derechos adquiridos · asignados técnicamente · utilizados realmente
    href: https://www.iso.org/standard/52293.html
  - label: Métricas
    wert: usuario · dispositivo · instancia · core/vCPU · capacidad · transacción · función
    href: https://www.iso.org/standard/52293.html
  - label: Punto de recuento
    wert: alcance + ancla de objeto + filtro de estado + ventana temporal + regla de agregación
    href: https://www.iso.org/standard/68531.html
  - label: ID de software
    wert: las etiquetas SWID estandarizan la identificación del producto, no automáticamente el derecho de uso
    href: https://www.iso.org/standard/65666.html
  - label: Licencias basadas en identidad
    wert: evaluar por separado la asignación directa y basada en grupos, la SKU y el plan de servicio
    href: https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails
  - label: HA y DR
    wert: evaluar nodos pasivos, Cold Standby e instancias de prueba y recuperación solo según los Terms concretos
    href: https://www.iso.org/standard/52293.html
  - label: Enforcement
    wert: la advertencia, el límite de funciones, la ausencia de nuevas asignaciones, el periodo de gracia o la interrupción del servicio dependen del producto
    href: https://www.cisco.com/site/us/en/buy/licensing/index.html
  - label: Funcionamiento sin conexión
    wert: documentar la duración de token/lease, el periodo de gracia, el reloj, el almacén de confianza y la ruta de recuperación
    href: https://www.cisco.com/site/us/en/buy/licensing/index.html
  - label: Código abierto
    wert: el uso gratuito no significa ausencia de obligaciones; comprobar el texto de la licencia y el escenario de distribución
    href: https://opensource.org/osd
  - label: Legible por máquina
    wert: las expresiones SPDX modelan licencias individuales, alternativas y combinadas
    href: https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/
  - label: Evidencia administrativa
    wert: exportar de forma reproducible entitlement · inventario · asignación · uso · excepción · tiempo · fuente
    href: https://www.iso.org/standard/68531.html
werbung:
  - newsletter
ctaThemen:
  - lizenzierung
  - messaging
  - itam
translationSourceHash: 4abc9d3d627b9f38adb1f2fcb541662b430f07694209251464f0a4b0f9c4351f
translationModel: gpt-5.6-terra
translatedAt: 2026-10-09T11:25:03.484Z
translationReview: required
---

# Licencias: derechos de uso, métricas y puntos técnicos de recuento

Las licencias de software vinculan un contrato legal de uso con objetos técnicamente medibles. Para los administradores es fundamental no confundir estos niveles. Una clave de licencia, una indicación en la nube o un contador interno de usuarios puede activar funciones y medir el uso; pero por sí solo no define lo que una organización tiene derecho a utilizar legalmente. ISO/IEC 19770-3 trata explícitamente los datos digitales de entitlement como una representación de derechos de uso y aclara que las condiciones de licencia originales prevalecen a efectos legales ([ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

Por tanto, la tarea operativa es: **conciliar de forma reproducible los derechos adquiridos, los derechos instalados o asignados técnicamente y el uso real**. Las desviaciones pueden ser sobreuso, costes no utilizados, filtros de directorio incorrectos, cuentas huérfanas, nodos pasivos de clúster, tokens vencidos o simplemente distintas definiciones de la misma palabra «usuario». El artículo describe la perspectiva técnica; la interpretación contractual y legal corresponde a compras, gestión de licencias y asesoramiento jurídico.

La explicación comienza con el derecho contractual de uso y sigue cómo el producto, la métrica y el punto técnico de recuento convierten ese derecho en consumo medido. Después se abordan la asignación, el enforcement, la auditoría, los casos especiales y la recuperación.

Una licencia es ante todo un derecho de uso; solo se hace visible técnicamente mediante asignación, uso medido y, en su caso, enforcement. El artículo distingue estos cuatro niveles antes de tratar las métricas de producto y las auditorías.

## Entitlement, asignación, uso y enforcement

Un balance de licencias limpio mantiene cuatro estados independientes:

1. **Entitlement:** ¿Qué derechos de uso se adquirieron con qué contrato, producto, métrica, alcance, período y derecho especial?
2. **Despliegue/asignación:** ¿En qué dispositivos, instancias, usuarios, tenants o funciones está instalado, activado o asignado el software?
3. **Uso:** ¿Qué objetos o funciones relevantes para las licencias se utilizaron realmente durante el período de medición acordado?
4. **Enforcement:** ¿Qué límite comprueba técnicamente el producto y cómo reacciona ante falta de conexión, vencimiento o exceso?

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-lizenzierung.svg?v=20260813" title="Interaktive Infografik: Lizenzarchitektur mit Vertrag und Entitlement, Inventar, Zuweisung und Nutzung, Metrik und Zählpunkt, Lizenzdienst, Enforcement, Ausnahmen sowie Audit- und Recoverypfad" loading="lazy">
  <a href="/images/kb-interaktiv-lizenzierung.svg?v=20260813">Abrir gráfico interactivo directamente</a>.
</iframe>

Un entitlement existente no demuestra una asignación correcta; una asignación no demuestra uso; un uso reducido no anula automáticamente una licencia de usuario nominada. A la inversa, un producto puede seguir funcionando técnicamente aunque haya vencido un derecho de suscripción o soporte. Por ello, el cumplimiento y la disponibilidad técnica son dos objetivos de control distintos.

ISO/IEC 19770-1 especifica requisitos para un sistema de gestión de activos de TI. ISO/IEC 19770-2 estandariza las etiquetas de identificación de software (SWID), que permiten identificar el software, pero según la norma no exigen una conciliación de entitlements. ISO/IEC 19770-3 define términos y un formato de transporte para entitlements y métricas asociadas. En conjunto proporcionan los niveles de datos **inventario**, **identidad de software** y **derecho de uso**, no un cálculo universal de licencias ([ISO/IEC 19770-1:2017](https://www.iso.org/standard/68531.html), [ISO/IEC 19770-2:2015](https://www.iso.org/standard/65666.html), [ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

## Métricas de licencia: ¿qué se cuenta?

Una métrica solo puede interpretarse junto con su contrato completo:

| Métrica | Posible ancla de recuento | Preguntas de administración |
|---|---|---|
| Usuario nominado | ID inmutable de persona/tenant | ¿se cuentan cuentas desactivadas, compartidas, externas, de servicio o de prueba; se puede reasignar? |
| Usuario/sesión concurrente | sesión activa o lease de checkout | ¿qué sesión empieza/termina, cómo se tratan los tiempos de espera, múltiples dispositivos y leases sin conexión? |
| Dispositivo | ID de hardware, dispositivo registrado, instancia de cliente | ¿se cuentan por separado VDI, dispositivo de sustitución, dispositivo compartido y agentes reinstalados? |
| Servidor/instancia/nodo | VM, host, appliance, nodo de clúster, instancia de contenedor | ¿se cuentan instancias pasivas, temporales, autoescaladas, de prueba o recuperación? |
| Procesador/core/vCPU | socket/core físico, vCPU asignada, cantidad mínima | ¿cómo se calculan Hyperthreading, afinidad, movimiento en clúster, tamaños de nube y paquetes mínimos? |
| Capacidad | buzones, almacenamiento, dominios, mensajes, rendimiento o registros | ¿se aplica pico, promedio, máximo mensual, provisioned o used; qué ventana temporal? |
| Función/edición | plan de servicio activado, módulo, límite de base de datos, API | ¿basta la instalación, activación, configuración o uso real? |
| Suscripción/consumo | asignación de SKU, créditos, requests, GB-mes | ¿cuándo se reserva, consume, factura posteriormente o devuelve? |

La unidad técnica no debe deducirse del nombre del producto. «Por core» puede significar cores físicos del host, vCPU de la VM o una métrica de core normalizada. «Usuario» puede significar persona física, cuenta activa, buzón, identidad con licencia o remitente emisor. «Instancia» puede contar en ejecución, instalada, registrada o por nodo de clúster. ISO/IEC 19770-3 estandariza una estructura de entitlement, pero no sustituye la definición concreta de los Product Terms y del pedido ([ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

La métrica por sí sola aún no indica dónde se cuenta. Solo el punto técnico de recuento explica por qué el portal del fabricante, el inventario local y el valor de facturación pueden diferir.

## El punto técnico de recuento

Cada métrica se documenta como una función de recuento reproducible:

**Contador = alcance × ancla de objeto × filtro de estado × ventana temporal × regla de agregación × excepciones**

- **Alcance:** organización, tenant, dominio, clúster, suscripción, ubicación o contrato.
- **Ancla de objeto:** ID de usuario inmutable, ID de dispositivo, ID de VM, número de serie de host, ID de SKU o hash; no solo nombre mostrado.
- **Filtro de estado:** enabled, assigned, provisioned, active, seen, mounted, running o consumed.
- **Ventana temporal:** fecha de corte, mes natural, pico, promedio, ventana móvil o año contractual.
- **Agregación:** distinct, suma, máximo, percentil 95, paquete mínimo o tramo.
- **Excepciones:** usuarios externos, cuentas del sistema, instancia DR pasiva, Trial, NFR o caso de uso libre por contrato.

El punto de recuento suele ser un **punto de transferencia**. Una importación LDAP puede contar todas las cuentas coincidentes, aunque nunca utilicen una función de correo. Un gateway puede aprender remitentes del tráfico y conservar así identidades eliminadas o técnicas. Una SKU en la nube puede asignarse directamente o de forma transitiva mediante un grupo. La API `licenseDetails` de Microsoft Graph proporciona licencias directas y heredadas mediante pertenencia a grupos, así como planes de servicio individuales y estados de aprovisionamiento; esto muestra por qué un booleano «tiene licencia» no basta para el análisis ([Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)).

## Arquitectura de un sistema de licencias

Los productos comerciales utilizan implementaciones diferentes, pero técnicamente pueden descomponerse en superficies de control. Este modelo es una síntesis operativa de los niveles de datos ISO/IEC 19770 y mecanismos documentados por fabricantes, no una arquitectura normativa universal:

| Superficie de control | Función | Área de fallo |
|---|---|---|
| Almacén de entitlement | contrato/SKU, cantidad, período, funciones, derechos especiales | pedido incorrecto, vencido, tenant/Smart Account incorrecto |
| Fuente de inventario/identidad | usuarios, dispositivos, instancias, cores, clústeres, ID de software | duplicado, objeto obsoleto, filtro o alcance incorrecto |
| Asignación | asigna un derecho a un objeto o plan de servicio | asignación directa frente a basada en grupos, error de aprovisionamiento |
| Medidor | recopila uso, sesiones, capacidad o heartbeats | tiempo, búfer sin conexión, muestreo, telemetría ausente |
| Evaluador | aplica métrica, pool, excepciones y período | la lógica contractual no coincide con la política técnica |
| Servicio de licencias | servidor local, API en la nube, token, lease, certificado o clave | DNS/TLS/proxy/reloj/confianza, fallo o límite de tasa |
| Enforcement | activa edición/función o limita el comportamiento | bloqueo duro, periodo de gracia, advertencia o fail-open/fail-closed |
| Exportación de evidencia | datos de auditoría, uso y asignación | rotación, protección de datos, historial ausente o exportación no reproducible |

Cisco Smart Licensing describe una gestión centralizada de cuentas y licencias; Microsoft Graph expone SKU adquiridas, asignaciones y planes de servicio mediante API. Estos sistemas convierten las licencias en una dependencia distribuida de identidad, servicio en la nube, red, TLS y tiempo. Un producto puede seguir procesando datos, pero no obtener una nueva licencia; otro puede bloquear funciones tras un período sin conexión. El comportamiento concreto debe tomarse de la documentación del fabricante y del contrato ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html), [Microsoft Graph – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0)).

## Licencias basadas en identidad

Con usuarios nominados, el directorio forma parte del sistema de licencias. Un filtro correcto no solo responde «personas en OU X», sino:

- qué atributo constituye el ancla de objeto inmutable;
- si cuentan las cuentas desactivadas, bloqueadas, eliminadas o aún no aprovisionadas;
- cómo se tratan los buzones compartidos/de recursos, cuentas de servicio, invitados, socios externos y cuentas de prueba;
- si se combinan las asignaciones directas y basadas en grupos;
- cuándo vuelve a estar disponible una licencia retirada;
- si es relevante el uso histórico o solo el inventario en la fecha de corte;
- qué atributos almacena el producto en caché localmente y cuándo los elimina.

LDAP proporciona entradas y atributos, pero no una definición universal de «persona con licencia». [LDAP](/kb/ldap) puede implementar técnicamente el alcance y el filtro; la métrica procede del contrato. Lo mismo se aplica a los directorios en la nube: SKU, plan de servicio, `assignedLicenses`, estado de aprovisionamiento y estado real de la carga de trabajo son datos diferentes. Microsoft Graph documenta `subscribedSku` como una suscripción comercial adquirida y `licenseDetails` por usuario ([Microsoft Graph – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0), [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)).

La limpieza de cuentas huérfanas no debe orientarse directamente a la presión de licencias. Primero se comprueban propietarios, retención, enrutamiento de correo, Legal Hold, dependencias de servicios y recuperación; después se desactiva o elimina la identidad de forma controlada. Un informe de licencias es una entrada para el ciclo de vida, no una orden de eliminación.

## Modelos de instancia, core y capacidad

La virtualización y los clústeres hacen insuficiente el nombre del servidor físico. Un inventario necesita host, VM/contenedor, vCPU asignadas, CPU/core físicos, clúster de hipervisor, reglas de movilidad, edición y rol. Si una VM puede moverse entre hosts, según el contrato puede ser relevante todo el alcance posible de hosts; una afinidad de CPU estricta puede demostrarse técnicamente, pero no está automáticamente reconocida por contrato.

Las ediciones vinculan el derecho de uso con límites técnicos. Microsoft documenta, por ejemplo, ediciones de Exchange Server que se diferencian, entre otras cosas, en el número de bases de datos montadas simultáneamente; las copias pasivas de bases de datos también pueden contar como bases de datos montadas. La Product Key establece la edición del servidor. Es un ejemplo de cómo se unen el enforcement y la arquitectura de capacidad, no un cálculo general de licencias de Exchange ([Microsoft – Exchange Server Editions and Versions](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deployment-ref/editions-and-versions), [Microsoft – Enter Exchange Product Key](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/enter-product-key)).

Las métricas de capacidad requieren una evolución temporal. Un valor puntual no muestra ni el pico mensual ni un exceso breve. Para mensajería, los contadores técnicos habituales incluyen buzones activos, remitentes internos, dominios, número diario de mensajes, throughput, almacenamiento y usuarios cifrados; solo el derecho del producto determina si son relevantes para las licencias. Por ello, los paneles almacenan el valor bruto, tiempo, alcance, fuente y regla de cálculo en lugar de solo un semáforo.

En cuanto se cuentan instancias o identidades, la alta disponibilidad, las pruebas y las migraciones también afectan a la cantidad de licencias. Los sistemas pasivos no son automáticamente gratuitos; prevalece el contrato respectivo.

## HA, recuperación ante desastres, pruebas y migración

Los nodos pasivos, Cold Standby, instancias de recuperación, laboratorio, prueba, formación y estados paralelos temporales durante una [migración](/kb/migration) son casos especiales típicos. Técnicamente, «pasivo» puede significar aun así que el software está instalado, procesa replicación, tiene bases de datos montadas o que se ha extraído una licencia en el servidor. Los términos contractuales deben asignarse a estados observables.

Para cada caso especial se documentan:

- número permitido y definición de instancias pasivas/frías;
- si se incluyen pruebas, cualificación de parches, restauración de copias de seguridad o pruebas de DR;
- cuánto tiempo se permite la operación simultánea antigua/nueva durante una migración;
- si una licencia es móvil y qué condiciones de reasignación/espera se aplican;
- si los derechos de nube y on-premises están vinculados;
- qué evidencia distingue un caso real de DR de una carga sostenida de producción;
- cómo funciona el servicio de licencias en una red de recuperación aislada.

El [concepto de backup y DR](/kb/backup-dr) incluye por tanto también archivos de licencia, datos de activación, servidores de licencias, tokens sin conexión, certificados, hora, DNS/proxy y contactos del fabricante. Una restauración técnicamente perfecta que no puede activar su edición o las funciones necesarias no cumple el objetivo de recuperación.

## Vencimiento, periodo de gracia y fallo del servicio de licencias

En modelos de suscripción, lease o nube, son relevantes al menos los siguientes temporizadores: fin del entitlement, vencimiento de token, heartbeat, duración de borrow/checkout, período sin conexión, Grace Period y validez del certificado. La perspectiva administrativa registra **hora absoluta, zona horaria, fuente de sincronización y última renovación correcta**. Un reloj incorrecto puede desencadenar un aparente vencimiento de licencia o impedir la validación de un token firmado.

El comportamiento al vencimiento depende del producto:

- solo advertencia o evento de cumplimiento;
- no hay nuevas asignaciones, el uso existente permanece;
- función prémium desactivada o regreso a una edición menor;
- cantidad limitada de nuevas sesiones/usuarios;
- funcionamiento de solo lectura;
- interrupción completa del servicio;
- periodo de gracia local cuando el servicio en la nube no es accesible.

Esta reacción se determina en un entorno de prueba o con una fuente explícita del fabricante, no se prueba durante la operación de producción. La monitorización advierte antes del temporizador operativo más temprano, no solo antes del fin del contrato. Cisco documenta mecanismos propios en línea/sin conexión y de cuenta para Smart Licensing; otros fabricantes utilizan servidores de licencias locales, archivos firmados, dongles USB, claves de producto o entitlements SaaS ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html)).

## Código abierto: derecho de uso sin contador técnico

El código abierto no significa simplemente código fuente visible. La Open Source Definition exige, entre otras cosas, redistribución libre, acceso al código fuente, obras derivadas y derechos neutrales respecto a la tecnología. Las licencias individuales imponen condiciones diferentes para modificación, distribución, avisos, entrega de código fuente o licencias de patentes ([Open Source Initiative – Open Source Definition](https://opensource.org/osd)).

Apache License 2.0 concede licencias de copyright y patentes bajo condiciones y exige, entre otras cosas, una copia de la licencia, indicaciones de modificación y conservación de determinados avisos en caso de redistribución. Un precio de descarga de cero no elimina estas obligaciones ([Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)). En conjuntos de componentes pueden aplicarse varias licencias simultánea o alternativamente.

Las expresiones de licencia SPDX modelan estos casos de forma legible por máquina: `AND` para obligaciones acumulativas, `OR` para una elección de licencia y `WITH` para una excepción. Una expresión SPDX identifica la situación de licencia declarada, pero no realiza una comprobación de compatibilidad legal ([SPDX Specification – License Expressions](https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/)). Para los administradores, los archivos de licencia y avisos, la SBOM/lista de componentes, el canal de distribución y las modificaciones propias forman parte de la evidencia de lanzamiento y archivo.

## Herramientas de administración para contadores reproducibles

Los ejemplos muestran consultas técnicas de inventario y evidencia. Si un campo es relevante para las licencias debe derivarse de los Terms. Las exportaciones pueden contener datos personales y requieren control de acceso, retención y limitación de finalidad.

### Recopilar inventario de CPU, cores y virtualización

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Hardwareinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Get-CimInstance`](https://learn.microsoft.com/en-us/powershell/module/cimcmdlets/get-ciminstance) lee datos de inventario CIM/WMI; [`lscpu`](https://man7.org/linux/man-pages/man1/lscpu.1.html) recopila datos de CPU, core, hilo, socket y NUMA. En una VM muestran principalmente la perspectiva del huésped. El alcance de host, el movimiento en clúster, el tipo de instancia de nube y los factores de core contractuales se documentan por separado.

### Contar objetos de directorio con alcance explícito

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Benutzerinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) y [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch) proporcionan anclas de objeto y atributos. Se documentan Search Base, filtro, paginación, atributos multivalor y objetos desactivados. La exportación no cuenta «personas sujetas a licencia» mientras la regla contractual no se haya asignado exactamente a estos campos.

### Leer SKU de nube y asignaciones de usuarios

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cloud-Lizenzinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --fail --silent --show-error \
  --header "Authorization: Bearer $GRAPH_ACCESS_TOKEN" \
  'https://graph.microsoft.com/v1.0/subscribedSkus?$select=skuId,skuPartNumber,consumedUnits,prepaidUnits,capabilityStatus'</code></pre>
  </div>
</div>

[`Get-MgSubscribedSku`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0), [`Get-MgUserLicenseDetail`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguserlicensedetail) y [`curl`](https://curl.se/docs/manpage.html) leen datos de Microsoft Graph. Los tokens no se almacenan en scripts, tickets ni historial de shell. `ConsumedUnits`, la asignación de usuarios, el aprovisionamiento del plan de servicio y la carga de trabajo realmente utilizada se evalúan por separado.

### Comprobar el servidor de licencias o el endpoint en la nube desde la red del producto

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenzdienst-Erreichbarkeit">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) comprueban la conexión DNS/TCP desde el origen elegido. Un resultado satisfactorio no prueba el proxy, [TLS](/kb/tls), el token, la asignación de cuenta ni la transacción de licencia. Los logs del producto y el estado del fabricante proporcionan la siguiente transición de estado.

### Proteger la exportación de auditoría contra modificaciones

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Auditexport">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) y [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) detectan cambios posteriores de bytes. Además, se documentan la hora de creación, el alcance de la consulta, la versión de herramienta/API, la consulta, la zona horaria, el exportador y el almacenamiento seguro. Un hash no confirma la integridad técnica.

### Comprobar eventos de licencia y activación durante el período

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Logprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">
  </code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">
  </code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent), [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) y [`grep`](https://www.gnu.org/software/grep/manual/grep.html) filtran eventos locales. Los logs del fabricante pueden usar códigos estructurados en lugar de mensajes de texto en inglés; se prefieren proveedor, ID de evento o campos definidos. El tiempo, la rotación y la copia centralizada de logs forman parte del hallazgo.

Del entitlement, el punto de recuento y el estado operativo surge la evidencia de auditoría. Debe mostrar de forma reproducible qué se compró, asignó, instaló y utilizó realmente.

## Evidencia de auditoría y operación

Una evidencia de licencia reproducible contiene:

- referencia de contrato/pedido, producto/SKU, métrica, cantidad, alcance, vigencia y derechos especiales;
- inventario de software y hardware con ID inmutables, edición, versión y relación de clúster;
- asignaciones directas, basadas en grupos y automáticas con fuente y estado de aprovisionamiento;
- valores de medición brutos y regla de cálculo por ventana temporal;
- excepciones con justificación, propietario, fecha de vencimiento y control técnico;
- HA/DR/prueba/migración como población propia;
- indicación del producto, exportación de API/CLI y cálculo independiente;
- tiempo, zona horaria, versión de consulta/herramienta, hash y almacenamiento protegido.

Las desviaciones se clasifican: **error de datos** (duplicados, cuentas antiguas), **error de modelo** (métrica incorrecta), **error de proceso** (licencia no retirada tras la salida), **error técnico** (sincronización/token/servidor) o **entitlement faltante**. Solo esta separación indica si la respuesta es limpieza, configuración, aclaración contractual, adquisición o respuesta a incidentes.

## Evolución técnica de las licencias

Las claves de producto locales y los dongles vincularon inicialmente el derecho de uso estrechamente a un equipo. Los servidores de licencias de red introdujeron pools compartidos y leases concurrentes; la virtualización requirió nuevas reglas de host, core y movilidad. Las suscripciones y SaaS trasladaron los entitlements a cuentas en la nube, grupos de identidad y planes de servicio. Cisco Smart Licensing y Microsoft Graph ilustran modelos centrales de cuenta/API, mientras ISO/IEC 19770 estandariza datos SWID y de entitlement ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html), [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails), [ISO/IEC 19770-2:2015](https://www.iso.org/standard/65666.html), [ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

Paralelamente, el código abierto hizo innecesaria la activación técnica para muchos componentes, pero no las condiciones de licencia. SPDX creó identificadores y expresiones breves legibles por máquina para cadenas de suministro de software. Así, la tarea administrativa pasó de «introducir una clave» a un problema de datos sobre contrato, identidad, activo, vigencia, telemetría y composición del software. El planteamiento contrario es el mismo que en la monitorización: anclas de objeto claras, ventanas temporales explícitas, datos brutos y cálculos reproducibles.

## Fuentes

- [ISO/IEC 19770-3:2016 – Entitlement Schema](https://www.iso.org/standard/52293.html)
- [ISO/IEC 19770-1:2017 – IT Asset Management Systems](https://www.iso.org/standard/68531.html)
- [ISO/IEC 19770-2:2015 – Software Identification Tag](https://www.iso.org/standard/65666.html)
- [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)
- [Cisco – Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html)
- [Microsoft Graph PowerShell – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0)
- [Microsoft – Exchange Server Editions and Versions](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deployment-ref/editions-and-versions)
- [Microsoft – Enter Exchange Server Product Key](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/enter-product-key)
- [Open Source Initiative – Open Source Definition](https://opensource.org/osd)
- [Apache Software Foundation – Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)
- [SPDX Specification 3.0.1 – License Expressions](https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/)
- [Microsoft Learn – Get-CimInstance](https://learn.microsoft.com/en-us/powershell/module/cimcmdlets/get-ciminstance)
- [Linux man-pages – lscpu](https://man7.org/linux/man-pages/man1/lscpu.1.html)
- [Microsoft Learn – Get-ADUser](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser)
- [OpenLDAP – ldapsearch](https://www.openldap.org/software/man.cgi?query=ldapsearch)
- [Microsoft Graph PowerShell – Get-MgUserLicenseDetail](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguserlicensedetail)
- [curl – command line manpage](https://curl.se/docs/manpage.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – SHA-2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
- [GNU Grep – Manual](https://www.gnu.org/software/grep/manual/grep.html)
