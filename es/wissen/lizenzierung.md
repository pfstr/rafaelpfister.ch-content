---
title: "Licencias: derechos de uso, métricas y puntos de recuento técnicos"
blatt: "lizenzierung"
description: "Licenciamiento técnico para administradores de infraestructura y mensajería: derechos, métricas de usuarios, dispositivos, instancias, cores y capacidad, ediciones, suscripciones, activación, servidores de licencias y API en la nube, fuentes de identidad, HA/DR, límites técnicos de enforcement, datos de auditoría, código abierto y verificación."
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
    wert: Las etiquetas SWID estandarizan la identificación de productos, no automáticamente el derecho de uso
    href: https://www.iso.org/standard/65666.html
  - label: Licenciamiento basado en identidad
    wert: evaluar por separado la asignación directa y basada en grupos, la SKU y el plan de servicio
    href: https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails
  - label: HA y DR
    wert: evaluar nodos pasivos, Cold Standby e instancias de prueba y recuperación únicamente según los Terms concretos
    href: https://www.iso.org/standard/52293.html
  - label: Enforcement
    wert: la advertencia, el límite de funciones, la ausencia de nuevas asignaciones, el periodo de gracia o la interrupción del servicio dependen del producto
    href: https://www.cisco.com/site/us/en/buy/licensing/index.html
  - label: Funcionamiento sin conexión
    wert: documentar la duración del token/lease, el periodo de gracia, el reloj, el almacén de confianza y la ruta de recuperación
    href: https://www.cisco.com/site/us/en/buy/licensing/index.html
  - label: Open Source
    wert: el uso gratuito no significa ausencia de obligaciones; revisar el texto de la licencia y el escenario de distribución
    href: https://opensource.org/osd
  - label: Legible por máquinas
    wert: Las expresiones SPDX modelan licencias individuales, alternativas y combinadas
    href: https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/
  - label: Evidencia de administración
    wert: exportar de forma reproducible entitlement · inventario · asignación · uso · excepción · tiempo · fuente
    href: https://www.iso.org/standard/68531.html
werbung:
  - newsletter
ctaThemen:
  - lizenzierung
  - messaging
  - itam
translationSourceHash: fb38ff7080a593642081c7fa017dd2d33735c4fd96cdf1ae256bebe4be37eb3d
translationModel: gpt-5.6-terra
translatedAt: 2026-11-06T03:42:22.420Z
translationReview: automatic
---

# Licencias: derechos de uso, métricas y puntos de recuento técnicos

La concesión de licencias de software vincula un contrato legal de uso con objetos técnicamente medibles. Para los administradores, es crucial no confundir estos niveles. Una clave de licencia, una visualización en la nube o un contador interno de usuarios pueden activar funciones y medir el uso; pero no definen por sí solos lo que una organización puede utilizar legalmente. ISO/IEC 19770-3 trata explícitamente los datos digitales de entitlement como representación de derechos de uso y aclara que las condiciones de licencia originales prevalecen a efectos legales ([ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

Por ello, la tarea operativa es **conciliar de forma reproducible los derechos adquiridos, los derechos instalados o asignados técnicamente y el uso real**. Las discrepancias pueden deberse a sobreutilización, costes sin aprovechar, filtros de directorio incorrectos, cuentas huérfanas, nodos pasivos de clúster, tokens expirados o simplemente definiciones distintas de la misma palabra «usuario». El artículo describe la perspectiva técnica; la interpretación contractual y legal corresponde a compras, gestión de licencias y asesoría jurídica.

La explicación comienza con el derecho de uso contractual y sigue cómo el producto, la métrica y el punto de recuento técnico convierten ese derecho en un consumo medido. A continuación se tratan la asignación, el enforcement, la auditoría, los casos especiales y la recuperación.

Una licencia es ante todo un derecho de uso; solo se hace técnicamente visible mediante la asignación, el uso medido y, en su caso, el enforcement. El artículo separa estos cuatro niveles antes de abordar las métricas de producto y las auditorías.

## Entitlement, asignación, uso y enforcement

Un balance de licencias correcto mantiene cuatro estados independientes:

1. **Entitlement:** ¿Qué derechos de uso se adquirieron con qué contrato, producto, métrica, alcance, periodo y derecho especial?
2. **Despliegue/asignación:** ¿En qué dispositivos, instancias, usuarios, tenants o funciones se ha instalado, activado o asignado el software?
3. **Uso:** ¿Qué objetos o funciones relevantes para la licencia se utilizaron realmente durante el periodo de medición acordado?
4. **Enforcement:** ¿Qué límite verifica técnicamente el producto y cómo reacciona ante falta de conexión, expiración o superación?

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-lizenzierung.svg?v=20260813" title="Interaktive Infografik: Lizenzarchitektur mit Vertrag und Entitlement, Inventar, Zuweisung und Nutzung, Metrik und Zählpunkt, Lizenzdienst, Enforcement, Ausnahmen sowie Audit- und Recoverypfad" loading="lazy">
  <a href="/images/kb-interaktiv-lizenzierung.svg?v=20260813">Abrir el gráfico interactivo directamente</a>.
</iframe>

Disponer de un entitlement no demuestra que la asignación sea correcta; una asignación no demuestra uso; un uso reducido no elimina automáticamente una licencia de usuario nominal. A la inversa, un producto puede seguir funcionando técnicamente aunque haya expirado un derecho de suscripción o soporte. Por tanto, el cumplimiento y la disponibilidad técnica son dos objetivos de control diferentes.

ISO/IEC 19770-1 especifica requisitos para un sistema de gestión de activos de TI. ISO/IEC 19770-2 estandariza las etiquetas de identificación de software (SWID), que permiten identificar el software, pero que según la norma no exigen una conciliación de entitlements. ISO/IEC 19770-3 define términos y un formato de transporte para entitlements y métricas relacionadas. En conjunto, proporcionan los niveles de datos **inventario**, **identidad de software** y **derecho de uso**, no un cálculo universal de licencias ([ISO/IEC 19770-1:2017](https://www.iso.org/standard/68531.html), [ISO/IEC 19770-2:2015](https://www.iso.org/standard/65666.html), [ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

## Métricas de licencia: ¿qué se cuenta?

Una métrica solo puede interpretarse junto con el contrato completo:

| Métrica | Posible ancla de recuento | Preguntas de administración |
|---|---|---|
| Usuario nominal | ID inmutable de persona/tenant | ¿se cuentan cuentas desactivadas, compartidas, externas, de servicio o de prueba; puede reasignarse? |
| Usuario/sesión concurrente | sesión activa o lease de checkout | ¿qué sesión comienza/termina, cómo se tratan los tiempos de espera, los dispositivos múltiples y los leases sin conexión? |
| Dispositivo | ID de hardware, dispositivo registrado, instancia de cliente | ¿se cuentan por separado VDI, dispositivos de sustitución, dispositivos compartidos y agentes recién instalados? |
| Servidor/instancia/nodo | VM, host, appliance, nodo de clúster, instancia de contenedor | ¿se cuentan instancias pasivas, temporales, autoescaladas, de prueba o de recuperación? |
| Procesador/core/vCPU | socket/core físico, vCPU asignada, número mínimo | ¿cómo se calculan Hyperthreading, afinidad, movimiento de clústeres, tamaños de nube y paquetes mínimos? |
| Capacidad | buzones, almacenamiento, dominios, mensajes, rendimiento o registros | ¿se aplica pico, media, máximo mensual, provisionado o usado; cuál es la ventana temporal? |
| Función/edición | plan de servicio, módulo, límite de base de datos o API activados | ¿basta con la instalación, activación, configuración o uso real? |
| Suscripción/consumo | asignación de SKU, créditos, requests, GB-meses | ¿cuándo se reserva, consume, factura posteriormente o devuelve? |

La unidad técnica no debe inferirse del nombre del producto. «Por core» puede referirse a los cores físicos del host, a las vCPU de la VM o a una métrica de core normalizada. «Usuario» puede ser una persona física, una cuenta activa, un buzón, una identidad licenciada o un remitente. «Instancia» puede contarse en ejecución, instalada, registrada o por nodo de clúster. ISO/IEC 19770-3 estandariza una estructura de entitlement, pero no sustituye la definición concreta de Product Terms y pedido ([ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

La métrica por sí sola aún no indica dónde se cuenta. Solo el punto de recuento técnico explica por qué el portal del fabricante, el inventario local y el valor facturado pueden diferir.

## El punto de recuento técnico

Cada métrica se documenta como una función de recuento reproducible:

**Contador = alcance × ancla de objeto × filtro de estado × ventana temporal × regla de agregación × excepciones**

- **Alcance:** organización, tenant, dominio, clúster, suscripción, ubicación o contrato.
- **Ancla de objeto:** ID de usuario inmutable, ID de dispositivo, ID de VM, serial del host, ID de SKU o hash; no solo el nombre mostrado.
- **Filtro de estado:** enabled, assigned, provisioned, active, seen, mounted, running o consumed.
- **Ventana temporal:** fecha de corte, mes natural, pico, media, ventana móvil o año contractual.
- **Agregación:** distinct, suma, máximo, percentil 95, paquete mínimo o tramo.
- **Excepciones:** usuarios externos, cuentas de sistema, instancia pasiva de DR, trial, NFR o caso de uso gratuito por contrato.

El punto de recuento suele ser un **punto de transferencia**. Una importación LDAP puede contar todas las cuentas coincidentes, aunque nunca utilicen una función de correo. Un gateway puede aprender remitentes del tráfico y, por tanto, conservar identidades eliminadas o técnicas. Una SKU en la nube puede asignarse directamente o indirectamente a través de un grupo. La API `licenseDetails` de Microsoft Graph proporciona licencias directas y heredadas a través de la pertenencia a grupos, además de planes de servicio individuales y estados de aprovisionamiento; esto muestra por qué un booleano «tiene licencia» no es suficiente para el análisis ([Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)).

## Arquitectura de un sistema de licencias

Los productos comerciales utilizan implementaciones diferentes, pero técnicamente pueden descomponerse en superficies de control. Este modelo es una síntesis operativa de los niveles de datos ISO/IEC 19770 y mecanismos documentados por fabricantes, no una arquitectura normativa universal:

| Superficie de control | Función | Área de fallo |
|---|---|---|
| Almacén de entitlements | contrato/SKU, cantidad, periodo, funciones, derechos especiales | pedido incorrecto, expirado, tenant/Smart Account incorrecto |
| Fuente de inventario/identidad | usuarios, dispositivos, instancias, cores, clústeres, ID de software | duplicado, objeto obsoleto, filtro o alcance incorrecto |
| Asignación | asigna un derecho a un objeto o plan de servicio | asignación directa frente a basada en grupos, error de aprovisionamiento |
| Medidor | recopila uso, sesiones, capacidad o heartbeats | tiempo, búfer sin conexión, muestreo, telemetría ausente |
| Evaluador | aplica métrica, pool, excepciones y periodo | la lógica contractual no coincide con la política técnica |
| Servicio de licencias | servidor local, API en la nube, token, lease, certificado o clave | DNS/TLS/proxy/reloj/confianza, caída o límite de tasa |
| Enforcement | activa edición/función o limita el comportamiento | bloqueo estricto, periodo de gracia, advertencia o fail-open/fail-closed |
| Exportación de evidencias | datos de auditoría, uso y asignación | rotación, protección de datos, historial ausente o exportación no reproducible |

Cisco Smart Licensing describe una gestión centralizada de cuentas y licencias; Microsoft Graph expone mediante API las SKU adquiridas, las asignaciones y los planes de servicio. Estos sistemas convierten la concesión de licencias en una dependencia distribuida de identidad, servicio en la nube, red, TLS y tiempo. Un producto puede seguir procesando datos, pero no obtener una nueva licencia; otro puede bloquear funciones tras un periodo sin conexión. El comportamiento concreto debe extraerse de la documentación del fabricante y del contrato ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html), [Microsoft Graph – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0)).

## Licenciamiento basado en identidad

Con usuarios nominales, el directorio forma parte del sistema de licencias. Un filtro correcto no solo responde «personas en OU X», sino también:

- qué atributo constituye el ancla inmutable del objeto;
- si cuentan las cuentas desactivadas, bloqueadas, eliminadas o aún no aprovisionadas;
- cómo se tratan los buzones compartidos/de recursos, las cuentas de servicio, invitados, socios externos y cuentas de prueba;
- si se combinan las asignaciones directas y basadas en grupos;
- cuándo vuelve a estar disponible una licencia retirada;
- si es relevante el uso histórico o solo el estado en la fecha de corte;
- qué atributos almacena el producto localmente en caché y cuándo los elimina.

LDAP proporciona entradas y atributos, pero no una definición universal de «persona licenciada». [LDAP](/kb/ldap) puede implementar técnicamente el alcance y el filtro; la métrica procede del contrato. Lo mismo ocurre en los directorios en la nube: la SKU, el plan de servicio, `assignedLicenses`, el estado de aprovisionamiento y el estado real de la carga de trabajo son datos distintos. Microsoft Graph documenta `subscribedSku` como suscripción comercial adquirida y `licenseDetails` por usuario ([Microsoft Graph – Get-MgSubscribedSku](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0), [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails)).

La limpieza de cuentas huérfanas no debe orientarse directamente a la presión de licencias. Primero se comprueban propietarios, retención, enrutamiento de correo, Legal Hold, dependencias de servicios y recuperación; después, la identidad se desactiva o elimina de forma controlada. Un informe de licencias es una entrada para el ciclo de vida, no una orden de eliminación.

## Modelos de instancia, core y capacidad

La virtualización y los clústeres hacen insuficiente el nombre físico del servidor. Un inventario necesita host, VM/contenedor, vCPU asignadas, CPU/core físico, clúster de hipervisor, reglas de movilidad, edición y función. Si una VM puede moverse entre hosts, según el contrato puede ser relevante todo el alcance posible de hosts; una afinidad de CPU estricta puede demostrarse técnicamente, pero no queda reconocida automáticamente por contrato.

Las ediciones vinculan el derecho de uso con límites técnicos. Microsoft documenta en Exchange, por ejemplo, ediciones que difieren, entre otras cosas, en el número de bases de datos montadas simultáneamente; las copias pasivas de bases de datos también pueden contar como bases de datos montadas. La Product Key establece la edición del servidor. Es un ejemplo de cómo coinciden enforcement y arquitectura de capacidad, no un cálculo general de licencias de Exchange ([Microsoft – Exchange Server Editions and Versions](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/deployment-ref/editions-and-versions), [Microsoft – Enter Exchange Product Key](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/enter-product-key)).

Las métricas de capacidad requieren una evolución temporal. Un valor momentáneo no muestra ni el pico mensual ni una sobreutilización de corta duración. Para mensajería, los contadores técnicos habituales incluyen especialmente buzones activos, remitentes internos, dominios, número diario de mensajes, rendimiento, almacenamiento y usuarios cifrados; solo los derechos de producto determinan si son relevantes para las licencias. Por ello, los paneles almacenan el valor bruto, la hora, el alcance, la fuente y la regla de cálculo, en lugar de limitarse a un semáforo.

En cuanto se cuentan instancias o identidades, la alta disponibilidad, las pruebas y las migraciones también afectan a la cantidad de licencias. Los sistemas pasivos no son automáticamente gratuitos; lo determinante es el contrato correspondiente.

## HA, recuperación ante desastres, pruebas y migración

Los nodos pasivos, Cold Standby, instancias de recuperación, laboratorio, pruebas, formación y estados paralelos temporales durante una [migración](/kb/migration) son casos especiales típicos. Técnicamente, «pasivo» puede seguir significando que el software está instalado, procesa replicación, tiene bases de datos montadas o realiza checkout de una licencia en el servidor. Los términos contractuales deben mapearse a estados observables.

Para cada caso especial se documentan:

- el número permitido y la definición de instancias pasivas/frías;
- si se incluyen pruebas, calificación de parches, restauración de backups o pruebas de DR;
- cuánto tiempo se permite el funcionamiento simultáneo del sistema antiguo y nuevo durante una migración;
- si una licencia es móvil y qué condiciones de reasignación/espera se aplican;
- si los derechos en la nube y on-premises están vinculados;
- qué evidencia distingue un caso de DR real de una carga productiva permanente;
- cómo funciona el servicio de licencias en una red de recuperación aislada.

El [concepto de backup y DR](/kb/backup-dr) también incluye archivos de licencia, datos de activación, servidores de licencias, tokens sin conexión, certificados, hora, DNS/proxy y contactos del fabricante. Una restauración técnicamente perfecta que no puede activar su edición o las funciones necesarias no cumple el objetivo de recuperación.

## Expiración, periodo de gracia y fallo del servicio de licencias

En los modelos de suscripción, lease o nube, son relevantes al menos los siguientes temporizadores: fin del entitlement, expiración del token, heartbeat, duración de borrow/checkout, periodo sin conexión, periodo de gracia y validez del certificado. La perspectiva administrativa registra **hora absoluta, zona horaria, fuente de sincronización y última renovación correcta**. Un reloj incorrecto puede provocar una aparente expiración de licencia o impedir la validación de un token firmado.

El comportamiento al expirar depende del producto:

- solo advertencia o evento de cumplimiento;
- no hay nuevas asignaciones, se mantiene el uso existente;
- función premium desactivada o retroceso a una edición inferior;
- número limitado de nuevas sesiones/usuarios;
- funcionamiento de solo lectura;
- interrupción completa del servicio;
- periodo de gracia local si el servicio en la nube no es accesible.

Esta reacción se determina en un entorno de pruebas o mediante una fuente explícita del fabricante, no se prueba durante la operación de producción. La monitorización advierte sobre el temporizador operativo más temprano, no solo sobre el fin del contrato. Cisco documenta mecanismos propios en línea/sin conexión y de cuenta para Smart Licensing; otros fabricantes utilizan servidores de licencias locales, archivos firmados, dongles USB, claves de producto o entitlements SaaS ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html)).

## Open Source: derecho de uso sin contador técnico

Open Source no significa simplemente código fuente visible. La Open Source Definition exige, entre otras cosas, libre redistribución, acceso al código fuente, trabajos derivados y derechos tecnológicamente neutrales. Las licencias individuales establecen distintas condiciones para la modificación, redistribución, avisos, provisión de código fuente o licencias de patentes ([Open Source Initiative – Open Source Definition](https://opensource.org/osd)).

La Apache License 2.0 concede licencias de copyright y patentes bajo condiciones y exige en la redistribución, entre otras cosas, una copia de la licencia, avisos de modificación y la conservación de determinados notices. Un precio de descarga de cero no elimina estas obligaciones ([Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0)). En conjuntos de componentes pueden aplicarse varias licencias simultánea o alternativamente.

Las expresiones de licencia SPDX modelan estos casos de forma legible por máquinas: `AND` para obligaciones acumulativas, `OR` para una elección de licencia y `WITH` para una excepción. Una expresión SPDX identifica la situación de licencia declarada, pero no realiza una comprobación de compatibilidad legal ([SPDX Specification – License Expressions](https://spdx.github.io/spdx-spec/v3.0.1/annexes/spdx-license-expressions/)). Para los administradores, los archivos de licencia y notices, la SBOM/lista de componentes, el canal de distribución y las modificaciones propias forman parte de la evidencia de versiones y archivo.

## Herramientas de administración para contadores reproducibles

Los ejemplos muestran consultas técnicas de inventario y evidencia. La relevancia de un campo para las licencias debe derivarse de los Terms. Las exportaciones pueden contener datos personales y requieren control de acceso, conservación y limitación de finalidad.

### Recopilar inventario de CPU, cores y virtualización

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Hardwareinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux y Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-CimInstance Win32_ComputerSystem |
  Select-Object Name,Manufacturer,Model,NumberOfProcessors,NumberOfLogicalProcessors,HypervisorPresent
Get-CimInstance Win32_Processor |
  Select-Object DeviceID,Name,SocketDesignation,NumberOfCores,NumberOfLogicalProcessors</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">lscpu --extended=CPU,NODE,SOCKET,CORE,ONLINE
lscpu</code></pre>
  </div>
</div>

[`Get-CimInstance`](https://learn.microsoft.com/en-us/powershell/module/cimcmdlets/get-ciminstance) lee datos de inventario CIM/WMI; [`lscpu`](https://man7.org/linux/man-pages/man1/lscpu.1.html) recopila datos de CPU, cores, threads, sockets y NUMA. En una VM muestran principalmente la perspectiva del invitado. El alcance del host, el movimiento de clúster, el tipo de instancia en la nube y los factores de core contractuales se documentan por separado.

### Contar objetos de directorio con alcance explícito

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Benutzerinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux y Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ADUser -SearchBase 'OU=Licensed,DC=example,DC=ch' -Filter * \
  -Properties ObjectGUID,Enabled,mail,employeeType |
  Select-Object ObjectGUID,SamAccountName,Enabled,mail,employeeType |
  Export-Csv .\\licensed-users.csv -NoTypeInformation -Encoding UTF8</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">ldapsearch -LLL -x -H ldaps://directory.example.ch \
  -b 'ou=Licensed,dc=example,dc=ch' \
  '(&(objectClass=person)(mail=*))' entryUUID uid mail employeeType</code></pre>
  </div>
</div>

[`Get-ADUser`](https://learn.microsoft.com/powershell/module/activedirectory/get-aduser) y [`ldapsearch`](https://www.openldap.org/software/man.cgi?query=ldapsearch) proporcionan anclas de objeto y atributos. Se documentan Search Base, filtro, paginación, atributos multivalor y objetos desactivados. La exportación no cuenta «personas sujetas a licencia» mientras la regla contractual no se haya mapeado exactamente a estos campos.

### Leer SKU de nube y asignaciones de usuarios

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Cloud-Lizenzinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux y Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-MgSubscribedSku -All |
  Select-Object SkuId,SkuPartNumber,ConsumedUnits,PrepaidUnits,CapabilityStatus
Get-MgUserLicenseDetail -UserId 'admin@example.ch' |
  Select-Object SkuId,SkuPartNumber,ServicePlans</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">curl --fail --silent --show-error \
  --header "Authorization: Bearer $GRAPH_ACCESS_TOKEN" \
  'https://graph.microsoft.com/v1.0/subscribedSkus?$select=skuId,skuPartNumber,consumedUnits,prepaidUnits,capabilityStatus'</code></pre>
  </div>
</div>

[`Get-MgSubscribedSku`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.identity.directorymanagement/get-mgsubscribedsku?view=graph-powershell-1.0), [`Get-MgUserLicenseDetail`](https://learn.microsoft.com/en-us/powershell/module/microsoft.graph.users/get-mguserlicensedetail) y [`curl`](https://curl.se/docs/manpage.html) leen datos de Microsoft Graph. Los tokens no se almacenan en scripts, tickets ni en el historial de shell. `ConsumedUnits`, la asignación de usuarios, el aprovisionamiento de planes de servicio y la carga de trabajo utilizada realmente se evalúan por separado.

### Comprobar el servidor de licencias o endpoint en la nube desde la red del producto

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenzdienst-Erreichbarkeit">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux y Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection license.example.ch -Port 443 -InformationLevel Detailed</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 license.example.ch 443</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) comprueban el establecimiento de DNS/TCP desde el origen seleccionado. Un resultado satisfactorio no demuestra ni el proxy, [TLS](/kb/tls), el token, la asignación de cuenta ni la transacción de licencia. Los logs del producto y el estado del fabricante proporcionan la siguiente transición de estado.

### Proteger contra cambios la exportación de auditoría

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Auditexport">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux y Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-FileHash .\\evidence\\license-inventory.csv -Algorithm SHA256
Get-FileHash .\\evidence\\entitlements.pdf -Algorithm SHA256</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sha256sum ./evidence/license-inventory.csv
sha256sum ./evidence/entitlements.pdf</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) y [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) detectan modificaciones posteriores de bytes. Además, se documentan la hora de creación, el alcance de la consulta, la versión de la herramienta/API, la query, la zona horaria, el exportador y el almacenamiento seguro. Un hash no confirma la integridad técnica del contenido.

### Comprobar eventos de licencia y activación durante el periodo

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Lizenz-Logprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux y Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$since = [datetime]'2026-08-01T00:00:00Z'
Get-WinEvent -FilterHashtable @{LogName='Application'; StartTime=$since} |
  Where-Object Message -Match 'licen[cs]e|activation|entitlement' |
  Select-Object TimeCreated,Id,ProviderName,LevelDisplayName,Message</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">journalctl --since '2026-08-01 00:00:00 UTC' \
  --output short-iso-precise --no-pager \
  | grep -Ei 'licen[cs]e|activation|entitlement'</code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent), [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) y [`grep`](https://www.gnu.org/software/grep/manual/grep.html) filtran eventos locales. Los logs de fabricantes pueden usar códigos estructurados en lugar de mensajes de texto en inglés; se prefieren el proveedor, el ID de evento o campos definidos. La hora, la rotación y la copia central de logs forman parte del hallazgo.

Del entitlement, el punto de recuento y el estado operativo surge la evidencia de auditoría. Debe mostrar de forma reproducible qué se adquirió, asignó, instaló y utilizó realmente.

## Evidencia de auditoría y operación

Una evidencia de licencia reproducible incluye:

- referencia de contrato/pedido, producto/SKU, métrica, cantidad, alcance, vigencia y derechos especiales;
- inventario de software y hardware con ID inmutables, edición, versión y relación de clúster;
- asignaciones directas, basadas en grupos y automáticas con fuente y estado de aprovisionamiento;
- valores de medición brutos y regla de cálculo para cada ventana temporal;
- excepciones con justificación, propietario, fecha de expiración y control técnico;
- HA/DR/pruebas/migración como población propia;
- visualización de producto, exportación de API/CLI y cálculo independiente;
- hora, zona horaria, versión de query/herramienta, hash y almacenamiento protegido.

Las discrepancias se clasifican como: **error de datos** (duplicados, cuentas antiguas), **error de modelo** (métrica incorrecta), **error de proceso** (licencia no retirada tras la salida), **error técnico** (sync/token/servidor) o **entitlement ausente**. Solo esta separación muestra si la respuesta es limpieza, configuración, aclaración contractual, adquisición o respuesta a incidentes.

## Evolución técnica de las licencias

Las claves de producto locales y los dongles vincularon inicialmente de forma estrecha el derecho de uso a un ordenador. Los servidores de licencias en red introdujeron pools compartidos y leases concurrentes; la virtualización requirió nuevas reglas de host, core y movilidad. Las suscripciones y SaaS trasladaron los entitlements a cuentas en la nube, grupos de identidad y planes de servicio. Cisco Smart Licensing y Microsoft Graph ilustran modelos centralizados de cuenta/API, mientras ISO/IEC 19770 estandariza datos SWID y de entitlement ([Cisco Licensing](https://www.cisco.com/site/us/en/buy/licensing/index.html), [Microsoft Graph – List licenseDetails](https://learn.microsoft.com/en-us/graph/api/user-list-licensedetails), [ISO/IEC 19770-2:2015](https://www.iso.org/standard/65666.html), [ISO/IEC 19770-3:2016](https://www.iso.org/standard/52293.html)).

En paralelo, Open Source hizo innecesaria la activación técnica para muchos componentes, pero no las condiciones de licencia. SPDX creó identificadores cortos y expresiones legibles por máquinas para cadenas de suministro de software. Así, la tarea de administración pasó de «introducir una clave» a un problema de datos sobre contrato, identidad, activo, vigencia, telemetría y composición de software. La respuesta es la misma que en la monitorización: anclas de objeto claras, ventanas temporales explícitas, datos brutos y cálculos reproducibles.

## Fuentes

- [ISO/IEC 19770-3:2016 – Esquema de entitlement](https://www.iso.org/standard/52293.html)
- [ISO/IEC 19770-1:2017 – Sistemas de gestión de activos de TI](https://www.iso.org/standard/68531.html)
- [ISO/IEC 19770-2:2015 – Etiqueta de identificación de software](https://www.iso.org/standard/65666.html)
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
