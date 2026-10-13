---
title: "Releases y actualizaciones: artefactos, despliegue y evidencia operativa"
blatt: "releases"
description: "Operación de releases y actualizaciones para sistemas de mensajería e infraestructura: versiones, canales, ciclo de vida, integridad de artefactos, priorización de vulnerabilidades, dependencias, despliegue escalonado, migración de estado, reversión, verificación y evolución técnica."
fakten:
  - label: Proceso
    wert: identificar · priorizar · obtener · instalar · verificar la instalación
    href: https://csrc.nist.gov/pubs/sp/800/40/r4/final
  - label: Unidad de release
    wert: artefacto + versión + plataforma de destino + metadatos + prueba de integridad + dependencias
    href: https://theupdateframework.io/specification/latest/
  - label: Versión
    wert: interpretar solo dentro del modelo de versionado documentado; SemVer no es universal
    href: https://semver.org/
  - label: Priorización
    wert: explotación · exposición · impacto · medida disponible · capacidad de recuperación
    href: https://www.cisa.gov/known-exploited-vulnerabilities-catalog
  - label: CVSS
    wert: gravedad técnica y métricas de contexto; no determina por sí solo la fecha de parcheo
    href: https://www.first.org/cvss/v4.0/specification-document
  - label: EPSS
    wert: probabilidad estadística de explotación observada; no evalúa daños
    href: https://www.first.org/epss/how-it-works.html
  - label: Integridad
    wert: verificar origen, hash, firma y, cuando corresponda, la procedencia de la compilación
    href: https://slsa.dev/spec/v1.2/
  - label: Despliegue
    wert: piloto/canary → oleadas pequeñas → oleadas amplias; cada oleada tiene criterios de interrupción
    href: https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/
  - label: Reversión
    wert: planificar por separado la reversión de código, datos y configuración
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
  - label: Aceptación de mensajería
    wert: entrante · saliente · Submission · autenticación · cola · políticas · almacén · monitorización
    href: https://www.nist.gov/publications/trustworthy-email-0
  - label: Límite de soporte
    wert: evaluar conjuntamente producto, rama, nivel de parche, plataforma y dependencias
    href: https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-server-update-faq
  - label: Evidencia operativa
    wert: versión objetivo/real, ID de artefacto, cambio, hora, resultado, desviación y responsable
    href: https://csrc.nist.gov/pubs/sp/800/40/r4/final
werbung:
  - tools
  - newsletter
ctaThemen:
  - releases
  - patch-management
  - messaging
translationSourceHash: f5adb284b43cb4bb0db285f310beabb3b7e8b6ac825ea0abf603e08062c0cbbb
translationModel: gpt-5.6-terra
translatedAt: 2026-10-12T11:44:09.708Z
translationReview: automatic
---

# Releases y actualizaciones: artefactos, despliegue y evidencia operativa

Un release es una unidad de software o firmware identificada y publicada; una actualización es la transición controlada de un sistema en funcionamiento de un estado aprobado a otro. Para los administradores, esto implica mucho más que el archivo de instalación: plataforma de destino, relación entre versión y soporte, dependencias, migraciones de configuración y datos, firma u otra prueba de procedencia, problemas conocidos, ruta de instalación, criterios de interrupción y verificación funcional. NIST define en consecuencia la gestión de parches empresariales como la identificación, priorización, obtención, instalación y verificación de parches, actualizaciones y mejoras ([NIST SP 800-40 Rev. 4](https://csrc.nist.gov/pubs/sp/800/40/r4/final)).

Los sistemas de mensajería intensifican esta tarea. Los listeners de Internet procesan entradas externas, las colas contienen responsabilidades aún no finalizadas, los clústeres distribuyen el estado y los usuarios esperan una entrega continua. Por ello, un instalador exitoso no demuestra ni un flujo de correo funcional ni un stack completo compatible. La operación de releases es un cambio de estado en una arquitectura de dependencias, con una base inicial medible, una propagación controlada y un estado final verificable.

La explicación comienza con el artefacto concreto y su procedencia. A continuación se tratan el tipo de release, la evaluación de riesgos, las pruebas y el despliegue escalonado; la interrupción, la reversión y la verificación completan la ruta de cambio.

Una actualización es el cambio controlado de un estado de software conocido a un nuevo artefacto. El proceso seguro empieza por el origen y la firma, pasa por la cualificación y el despliegue escalonado, y solo finaliza con la aceptación técnica y funcional.

## Objeto de release y cadena de confianza

Una aprobación sólida describe el **objeto de release exacto**. El nombre comercial «Versión 16» no basta si la edición, la compilación, el nivel de hotfix, la plataforma, la arquitectura o el digest de imagen determinan el binario que realmente se ejecuta. Tampoco un nombre de archivo es una identidad: en un modelo de versionado limpio, un artefacto ya publicado no debe sustituirse silenciosamente. Semantic Versioning exige, por ejemplo, que el contenido de una versión publicada permanezca inalterado; sin embargo, esta regla solo se aplica si el fabricante garantiza realmente SemVer ([Semantic Versioning 2.0.0](https://semver.org/)).

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-releases.svg?v=20260813" title="Interaktive Infografik: Releaseobjekt, Vertrauenskette, Risikopriorisierung, Betriebsmodelle, gestufter Rollout, Abbruch, Rollback und Verifikation" loading="lazy">
  <a href="/images/kb-interaktiv-releases.svg?v=20260813">Abrir directamente el gráfico interactivo</a>.
</iframe>

Un objeto de release incluye como mínimo:

| Componente | Pregunta del administrador | Evidencia |
|---|---|---|
| Artefacto | ¿Qué bytes se instalan o inician? | Hash, ID de paquete, digest de imagen o identificador de firmware |
| Fabricante y procedencia | ¿De qué canal aprobado procede? | Repositorio HTTPS, metadatos firmados, portal del fabricante, cadena de transferencia sin conexión |
| Versión y destino | ¿Para qué producto, edición, sistema operativo, arquitectura y release de origen? | Notas de release, matriz de compatibilidad, ruta de actualización |
| Integridad y autenticidad | ¿El artefacto está inalterado y autorizado por el editor esperado? | Firma, verificación de certificado/clave, hash de una fuente autenticada |
| Contenido y procedencia | ¿Qué componentes y qué compilación lo generaron? | SBOM, procedencia de compilación, attestación, identidad del sistema de compilación |
| Cambio de estado | ¿Qué esquemas, índices, colas, configuraciones o claves cambian? | Plan de migración, pasos irreversibles, requisitos de almacenamiento y duración |
| Contrato operativo | ¿Qué servicios, puertos, API, valores predeterminados y dependencias cambian de comportamiento? | Cambios incompatibles, obsolescencias, problemas conocidos, nuevos requisitos mínimos |

The Update Framework (TUF) modela la distribución segura de actualizaciones con roles firmados para metadatos Root, Targets, Snapshot y Timestamp. Su objetivo es limitar, entre otras cosas, las claves de repositorio comprometidas, la reversión a estados antiguos y los metadatos congelados. SLSA describe la procedencia de compilación como información verificable sobre cómo, dónde y a partir de qué entradas se creó un artefacto. Ambos enfoques muestran que un hash solo confirma la igualdad con un valor de referencia; únicamente el origen fiable de ese valor responde a la cuestión de la autenticidad ([TUF Specification](https://theupdateframework.io/specification/latest/), [SLSA Specification 1.2](https://slsa.dev/spec/v1.2/)).

## Stack tecnológico de la ruta de release

La operación de releases es en sí misma un stack tecnológico. Conecta el sistema de código fuente y compilación, el formato de artefacto o paquete, los procedimientos de firma y procedencia, el repositorio o registro, el canal de distribución, el instalador u orquestador, la migración de configuración y esquema, así como la telemetría y el control de reversión. TUF describe los roles de metadatos firmados de un repositorio; SLSA describe la procedencia entre el estado fuente, la plataforma de compilación y el artefacto ([TUF Specification](https://theupdateframework.io/specification/latest/), [SLSA Specification 1.2](https://slsa.dev/spec/v1.2/)).

Al stack propio del producto de destino se suman el sistema operativo, la arquitectura de CPU, el runtime, la base de datos, el directorio, el proxy, las extensiones y los clientes. Un release solo está cualificado cuando ambos stacks son compatibles. Una imagen de contenedor firmada de forma válida puede ser inutilizable en la arquitectura equivocada; un gestor de paquetes exitoso tampoco puede deshacer una migración de base de datos no compatible.

## Versiones, canales y límites de soporte

Los números de versión son nombres dentro de un contrato del fabricante, no una escala universal de riesgo. En SemVer, `MAJOR.MINOR.PATCH` indica cambios de API incompatibles, funcionalidad compatible con versiones anteriores y corrección de errores compatible con versiones anteriores. Sin embargo, los fabricantes pueden utilizar modelos de calendario, compilación, firmware, actualizaciones acumulativas, funciones o suscripción. De `3.9.1` no se deduce sin documentación ni que `3.10.0` sea compatible ni que un parche sea pequeño o de bajo riesgo ([Semantic Versioning 2.0.0](https://semver.org/)).

También «stable», «LTS», «extended», «preview», «early access» y «general availability» son canales de release específicos del fabricante. Por ello, un inventario administrativo no solo almacena la versión en ejecución, sino también el canal, el fin de soporte, las rutas compatibles de predecesores/sucesores y la matriz de plataformas. Lo decisivo es el **conjunto de soporte** formado por rama de producto, nivel de parche, sistema operativo, runtime, base de datos, directorio, navegador/cliente, extensiones y, cuando corresponda, modelo de hardware.

El modelo de mantenimiento de Exchange de Microsoft ilustra esta multidimensionalidad: las Cumulative Updates, Security Updates y Hotfix Updates tienen requisitos y relaciones diferentes; las Security Updates son específicas de cada CU y, además, pueden requerirse medidas manuales. Por tanto, la regla administrativa estática no es «Exchange se parchea mensualmente», sino: para cada servidor se comprueban conjuntamente el estado compatible de CU/SU, el estado de Windows, las Management Tools y las acciones adicionales correspondientes al aviso concreto ([Microsoft – Exchange Server Update FAQ](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-server-update-faq)).

Cuando están definidos el artefacto, el canal y el estado de soporte, se evalúa el riesgo de cambio. Un número de versión alto por sí solo no indica ni urgencia ni compatibilidad.

## Priorización: riesgo en lugar de número de versión

No toda versión nueva es urgente, y no toda vulnerabilidad crítica puede esperar a la siguiente ventana rutinaria. La priorización combina varias señales independientes:

1. **Evidencia de explotación:** CISA mantiene el Known Exploited Vulnerabilities Catalog como lista autorizada de vulnerabilidades explotadas activamente y recomienda utilizarlo como entrada para la priorización ([CISA KEV Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)).
2. **Gravedad técnica:** CVSS v4.0 describe métricas Base, Threat, Environmental y Supplemental. La puntuación no constituye una declaración completa sobre el activo propio ni sobre una fecha de parcheo ([FIRST – CVSS v4.0](https://www.first.org/cvss/v4.0/specification-document)).
3. **Probabilidad de explotación:** EPSS estima la probabilidad de que una CVE publicada se explote realmente dentro de un intervalo de tiempo. EPSS no evalúa el daño potencial ni sustituye el conocimiento de los activos ([FIRST – EPSS Model](https://www.first.org/epss/how-it-works.html)).
4. **Exposición propia:** los listeners de Internet, una interfaz de administración accesible, la autenticación existente, la función activa, la versión, los controles compensatorios y el valor de los datos determinan si la ruta descrita existe en el sistema propio.
5. **Impacto operativo:** la interrupción, el crecimiento de la cola, la migración de datos, la pérdida del clúster o las dependencias no probadas pueden ser provocados por la propia actualización.
6. **Capacidad de recuperación:** una reversión probada, nodos de sustitución y un [modelo de backup/DR](/kb/backup-dr) actualizado permiten una velocidad distinta que una actualización irreversible de esquema sin prueba de recuperación.

El resultado es un plazo justificado con responsable, no solo «CVSS 9.8». Una vulnerabilidad explotada activamente en una pasarela de Internet puede exigir medidas inmediatas, reducción de la exposición y una ventana de emergencia. Un componente local no utilizado en un sistema aislado puede planificarse en una oleada regular con una compensación documentada. Las excepciones reciben una fecha de vencimiento; de lo contrario, se convierten en desviaciones permanentes invisibles.

## Enfoques arquitectónicos para actualizaciones

La reversión y el alcance de la interrupción dependen del enfoque arquitectónico:

| Modelo operativo | Cambio de estado | Patrón de despliegue | Límite central |
|---|---|---|---|
| Servidor o appliance in situ | Binarios, paquetes y, a menudo, configuración en el mismo nodo | Modo de mantenimiento, actualizar nodo, comprobar, reanudar | una actualización parcial o un instalador fallido puede dejar el nodo en un estado incierto |
| Clúster con roles con estado | Nodo más estado de replicación/quórum | mover roles, aislar un nodo, actualizar, resincronizar y luego la siguiente oleada | funcionamiento con versiones mixtas, quórum, carga de reconstrucción y orden de versiones |
| Imagen inmutable / contenedor | una nueva imagen sustituye la instancia antigua; configuración y datos están fuera | Canary, RollingUpdate, Blue/Green | la reversión de la imagen no ayuda frente a una migración de datos irreversible |
| SaaS / servicio gestionado | el fabricante despliega el servicio; el cliente controla la configuración y las integraciones | oleadas anunciadas, Feature Flags, compatibilidad de tenant/API | el cliente no puede revertir binarios; el contrato, la telemetría y las pruebas de integración son fundamentales |
| Firmware / appliance de hardware | firmware, particiones de arranque, controladores y, en su caso, formato de datos | primero nodo en espera/HA o banco dual, luego lado activo | la capacidad de arranque, el modelo de hardware y la ruta de degradación son específicos del fabricante |

Los Deployments de Kubernetes muestran explícitamente el modelo Rolling: `maxUnavailable` y `maxSurge` limitan respectivamente los Pods ausentes y adicionales durante la sustitución; el despliegue puede observarse, pausarse y revertirse a una revisión guardada. Esto solo garantiza el estado del controlador. Una migración de base de datos, un estado de cola externo o clientes incompatibles quedan fuera de esta reversión ([Kubernetes – Update a Deployment Without Downtime](https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/)).

Para los clústeres de mensajería, «primero el nodo en espera» también es correcto solo si el fabricante admite esta ruta. Los clústeres de configuración peer-to-peer, los pools de balanceadores de carga, la replicación de bases de datos y la alta disponibilidad de colas son mecanismos distintos. Antes de la actualización debe estar claro qué estados se replican, qué versiones pueden trabajar juntas temporalmente y si un nodo actualizado escribe datos en un formato que el nodo antiguo aún puede leer.

## Notas de release como contrato técnico

Las notas de release no se revisan superficialmente por palabras clave, sino que se traducen a una matriz local de cambios:

- **Seguridad:** CVE, componente/función afectada, vector de ataque, privilegio, explotación conocida, mitigación y acción adicional.
- **Requisitos previos:** versiones de origen permitidas, releases intermedios, espacio libre, runtime, sistema operativo, esquema, licencia y modelo de hardware.
- **Comportamiento:** nuevos valores predeterminados, puertos modificados, cifrados/protocolos, campos de API/CLI, autenticación, certificados, enrutamiento de correo y límites.
- **Eliminación:** obsolescencias, funciones eliminadas, algoritmos antiguos, plataformas retiradas o interfaces de gestión descontinuadas.
- **Migración:** duración, bloqueos, creación de cola/índice, formato de datos, compatibilidad con versiones anteriores y trabajos posteriores necesarios.
- **Problemas conocidos:** desencadenante, detección, solución alternativa, objetivo de corrección e impacto sobre la reversión o HA.
- **Ruta de instalación:** orden, reinicios, roles de nodo, comprobaciones de estado y pasos manuales posteriores.

La documentación del fabricante es la fuente primaria; los foros y los informes de experiencia pueden proporcionar hipótesis, pero no sustituyen una relación de actualización compatible. NIST SSDF exige archivar el software aprobado y sus componentes, proteger su integridad y proporcionar información para subsanar vulnerabilidades. Para los operadores, esto significa conservar de forma referenciable las notas de release, los artefactos, la información de hash/firma, la decisión de aprobación y los resultados de las pruebas locales ([NIST SP 800-218](https://csrc.nist.gov/pubs/sp/800/218/final)).

Las notas de release describen los cambios esperados; la cualificación los comprueba frente a la topología propia, el volumen de datos, las integraciones y las posibilidades de reversión.

## Cualificación antes de la ventana de mantenimiento

El entorno de pruebas no debe tener el mismo tamaño, pero sí ser representativo en las **propiedades relevantes para el riesgo**: misma edición de producto, autenticación, cadenas de certificados, versión de directorio y base de datos, reglas de flujo de correo, extensiones, tamaños de mensajes, protocolos y rutas de actualización. Un laboratorio recién instalado no prueba un error de actualización que solo aparece con una configuración desarrollada durante años.

Antes de la aprobación se comprueba como mínimo:

1. descarga desde una fuente aprobada y prueba de integridad/firma;
2. requisitos previos y ruta intermedia documentada;
3. copia de seguridad de configuración y datos con una ruta de restauración conocida;
4. actualización de una copia representativa o de un nodo piloto;
5. transacciones funcionales, incluidos casos negativos y decisiones de política;
6. rendimiento, cola, replicación, índice/reconstrucción y crecimiento de almacenamiento;
7. efecto de la monitorización y las alertas durante la interrupción, degradación y retorno;
8. reversión o recuperación hasta el punto de decisión definido;
9. duración con reserva para interrupción, recuperación y verificación.

Una instantánea no es una reversión universal. En sistemas distribuidos, la base de datos, la cola, el Blob Store, el directorio, los certificados y los pares externos pueden seguir funcionando después de la instantánea. Restablecer un único nodo puede generar un estado incoherente o ya procesado de nuevo. El [modelo de backup y DR](/kb/backup-dr) debe identificar el punto de consistencia común y los límites de responsabilidad.

## Despliegue escalonado y criterios de interrupción

Un despliegue controlado limita el radio de impacto mediante oleadas:

1. **Piloto/Canary:** unidad pequeña, observable y representativa sin función única.
2. **Oleada temprana:** varios sistemas o usuarios para hacer visibles dependencias menos frecuentes.
3. **Oleadas amplias:** capacidad, regiones, roles o tenants en lotes definidos.
4. **Observación posterior:** como mínimo durante los ciclos relevantes de reintento, caché, cola, certificados y carga diaria.

Cada oleada tiene criterios **Go, Hold y Abort** definidos previamente: tasa de errores, latencia, antigüedad/profundidad de la cola, distribución SMTP 4xx/5xx, errores de autenticación, retraso de replicación, recursos, crash loops, integridad de datos e incidencias de soporte. «No hay tickets» no es una comprobación de estado suficiente. La siguiente oleada solo comienza cuando se cumplen el período de medición y los criterios.

Ante un aborto no se revierte automáticamente. Si una reversión es más peligrosa que mantener el estado, por ejemplo, después de un cambio de esquema hacia adelante, la respuesta segura puede ser: pausar el despliegue, retirar el tráfico del nodo afectado, proteger los datos y seguir la ruta del fabricante. La matriz de decisión se crea antes de la ventana, no se improvisa bajo presión de tiempo.

Tras un cambio puramente binario, una reversión puede ser sencilla. En cuanto se modifican el esquema, el índice o los datos almacenados, debe aclararse de antemano si el retorno sigue siendo técnicamente posible.

## Reversión, avance y migración de datos

«Volver a la versión anterior» se descompone en al menos cuatro cuestiones de retorno independientes:

- **Código:** ¿Está disponible y firmado el paquete, la imagen o la imagen de firmware anterior? ¿El fabricante admite la degradación?
- **Configuración:** ¿Se modificó el formato y la versión anterior entiende la nueva configuración?
- **Datos:** ¿Se migró hacia adelante un esquema o índice? ¿Puede el software antiguo leer esos datos o es necesaria una restauración?
- **Efecto externo:** ¿Se aceptaron mensajes, se emitieron tokens, se modificó DNS, se rotaron certificados o otros sistemas utilizaron las API?

En frontends sin estado, una reversión de imagen suele ser rápida. En una plataforma de correo, el mismo paso puede procesar mensajes por duplicado o perder estados aceptados recientemente. Por eso existe un **punto de no retorno**: hasta entonces se ha probado una reversión técnica; después se prevé avanzar, restaurar o usar un modo de compatibilidad definido.

Además, la reversión no sustituye el análisis de causa raíz. Tras un retroceso, un artefacto defectuoso, un requisito previo incompleto, una desviación local, un límite de capacidad y un error de producto desconocido siguen siendo hipótesis distintas. El hallazgo del release registra la versión exacta, el host, la fase, el código de error, la hora y los registros asegurados, y sigue la [metodología de troubleshooting](/kb/troubleshooting).

## Herramientas de administración para inventario y verificación

Los comandos proporcionan elementos para una evidencia de cambio; no sustituyen los comprobadores de estado del fabricante ni las instrucciones de actualización específicas del producto. La salida se guarda antes y después de la actualización con hora, host e ID de cambio. Los secretos o configuraciones completas no deben incluirse sin protección en los tickets.

### Registrar el inventario de plataforma y parches

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Release-Inventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ComputerInfo -Property WindowsProductName,WindowsVersion,OsBuildNumber,OsArchitecture
Get-HotFix | Sort-Object InstalledOn,HotFixID</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">uname -a
cat /etc/os-release
rpm -qa --qf '%{NAME}\t%{EPOCHNUM}:%{VERSION}-%{RELEASE}\t%{ARCH}\n' 2&gt;/dev/null \
  || dpkg-query -W -f='${binary:Package}\t${Version}\n'</code></pre>
  </div>
</div>

[`Get-ComputerInfo`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-computerinfo) y [`Get-HotFix`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-hotfix) registran información de Windows y de actualizaciones registradas. [`uname`](https://www.gnu.org/software/coreutils/manual/html_node/uname-invocation.html), [`os-release`](https://www.freedesktop.org/software/systemd/man/latest/os-release.html), [`rpm`](https://rpm.org/docs/6.0.x/man/rpm.8) y [`dpkg-query`](https://manpages.debian.org/stable/dpkg/dpkg-query.1.en.html) proporcionan datos de sistema operativo y paquetes en Linux. La compilación del producto, el firmware de appliance, el digest de contenedor y las extensiones deben registrarse además con las herramientas respectivas del fabricante.

### Comprobar el hash y la firma del artefacto

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Artefaktprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-FileHash C:\\Staging\\update.msi -Algorithm SHA256
Get-AuthenticodeSignature C:\\Staging\\update.msi |
  Format-List Status,StatusMessage,SignerCertificate,TimeStamperCertificate</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">sha256sum update.tar.gz
gpgv release-signature.asc release-manifest.txt</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash) y [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) comparan bytes con un valor de referencia autenticado. [`Get-AuthenticodeSignature`](https://learn.microsoft.com/powershell/module/microsoft.powershell.security/get-authenticodesignature) y [`gpgv`](https://www.gnupg.org/documentation/manuals/gnupg/gpgv.html) verifican firmas frente al ancla de confianza configurada en cada caso. Un estado «válido» solo responde a la cuestión del editor si se esperan el certificado, la clave, el nombre, el período de validez y la vía de obtención.

### Comparar servicios y listeners antes y después del despliegue

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Dienst- und Listenerprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Service | Sort-Object Name | Select-Object Name,Status,StartType
Get-NetTCPConnection -State Listen |
  Sort-Object LocalPort | Select-Object LocalAddress,LocalPort,OwningProcess</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">systemctl --failed --no-pager
systemctl list-units --type=service --state=running --no-pager
ss -lntup</code></pre>
  </div>
</div>

[`Get-Service`](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-service), [`Get-NetTCPConnection`](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection), [`systemctl`](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html) y [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) muestran el estado local de servicios y listeners. Un proceso en ejecución aún no es una comprobación de estado funcional; la dirección de enlace, el puerto, el proceso, TLS, la respuesta de protocolo y las dependencias posteriores se comprueban por separado.

### Comprobar la accesibilidad de un endpoint representativo

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Release-Smoke-Test">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Test-NetConnection mx1.example.ch -Port 25 -InformationLevel Detailed
Test-NetConnection admin.example.ch -Port 443 -InformationLevel Detailed</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">nc -vz -w 5 mx1.example.ch 25
nc -vz -w 5 admin.example.ch 443</code></pre>
  </div>
</div>

[`Test-NetConnection`](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection) y [`nc`](https://man.openbsd.org/nc) demuestran el establecimiento de TCP desde el origen seleccionado. A continuación se realizan pruebas de [TLS](/kb/tls)-, [SMTP](/kb/smtp)-, autenticación y flujo de correo funcional; un puerto abierto no confirma un servicio operativo.

### Asegurar errores y advertencias en la ventana de cambio

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Release-Logprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">$since = [datetime]'2026-08-08T18:00:00Z'
Get-WinEvent -FilterHashtable @{LogName='System'; StartTime=$since; Level=1,2,3} |
  Select-Object TimeCreated,Id,ProviderName,LevelDisplayName,Message</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">journalctl --since '2026-08-08 18:00:00 UTC' \
  --priority warning --output short-iso-precise --no-pager</code></pre>
  </div>
</div>

[`Get-WinEvent`](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent) y [`journalctl`](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html) limitan el hallazgo a la ventana de mantenimiento. Los registros del instalador, producto, clúster y base de datos se aseguran además con fuente y base temporal exactas. La comprobación no busca solo nuevos errores, sino también eventos esperados ausentes, reinicios repetidos y reintentos crecientes.

Un estado de proceso sin incidencias no basta en mensajería. La aceptación, la cola, la política, la entrega saliente, el acceso de clientes y el seguimiento deben comprobarse a través de la ruta real del mensaje.

## Aceptación específica de mensajería

Después de la actualización no se envía únicamente un correo de prueba. La aceptación refleja los roles de la plataforma:

- **Relay entrante:** selección DNS/MX externa, TCP/TLS, comprobación de destinatarios, aceptación después de DATA, política de contenido y dominio, cola y entrega definitiva.
- **Relay saliente:** selección de conector/smarthost, IP de origen, política TLS, [SPF/DKIM/DMARC](/kb/mail-auth), reintento de cola y aceptación por un destinatario controlado.
- **Submission y acceso:** autenticación, autorización, IMAP/HTTP/API, Send-as/Delegate y clientes móviles/heredados en la medida en que sean compatibles.
- **Identidad:** [LDAP](/kb/ldap)-Bind y -búsqueda, resolución de grupos/atributos, [Kerberos](/kb/kerberos), tokens y certificados.
- **Política y criptografía:** malware/DLP, cuarentena, disclaimer, enrutamiento, [S/MIME/OpenPGP/Portal](/kb/verschluesselung) y acceso a claves.
- **Estado y HA:** cola, base de datos, almacén de objetos, replicación, índice, agente de backup, failover y retorno del nodo actualizado.
- **Administración:** GUI, API, CLI, roles, registro de auditoría, exportación de configuración y monitorización.

Cada prueba tiene una respuesta esperada y real, un ID de correlación, hora, origen y destino. Una comprobación sintética en verde y un pequeño piloto real se complementan: la comprobación sintética es reproducible; el piloto cubre combinaciones imprevistas de clientes, políticas y carga. La separación entre transporte, autenticación de dominio y mecanismos de contenido sigue la arquitectura técnica de correo electrónico de NIST SP 800-177 Rev. 1 ([NIST – Trustworthy Email](https://www.nist.gov/publications/trustworthy-email-0)).

## Evolución técnica de la operación de releases

Antes, los estados de sistemas se distribuían a menudo como medios completos o parches de fabricante aplicados manualmente. Los gestores de paquetes, los repositorios firmados y los modelos de mantenimiento acumulativo automatizaron las dependencias y la distribución; las máquinas virtuales, imágenes y contenedores desplazaron el foco hacia la sustitución reproducible de instancias. CI/CD redujo el tiempo entre la compilación y producción, mientras que TUF, SLSA y SSDF modelan explícitamente la cadena de confianza del código fuente, el sistema de compilación, los metadatos y el artefacto ([TUF Specification](https://theupdateframework.io/specification/latest/), [SLSA Specification 1.2](https://slsa.dev/spec/v1.2/), [NIST SP 800-218](https://csrc.nist.gov/pubs/sp/800/218/final)).

La consecuencia operativa no es «instalar todo automáticamente», sino **identificar cada cambio de estado de forma legible por máquina, cualificarlo de forma reproducible, desplegarlo de manera limitada y verificarlo funcionalmente**. La automatización reduce las desviaciones manuales; sin oleadas ni lógica de interrupción, también aumenta la velocidad de un error. Por ello, las plataformas modernas de releases necesitan los mismos controles clásicos —inventario, aprobación, ventana de mantenimiento y recuperación— complementados por procedencia de artefactos, estado objetivo declarativo, telemetría y entrega progresiva.

## Fuentes

- [NIST SP 800-34 Rev. 1 – Contingency Planning Guide](https://csrc.nist.gov/pubs/sp/800/34/r1/final)
- [NIST SP 800-177 Rev. 1 – Trustworthy Email](https://www.nist.gov/publications/trustworthy-email-0)
- [NIST – SP 800-40 Rev. 4, Enterprise Patch Management](https://csrc.nist.gov/pubs/sp/800/40/r4/final)
- [Semantic Versioning 2.0.0](https://semver.org/)
- [The Update Framework – Specification](https://theupdateframework.io/specification/latest/)
- [SLSA – Specification 1.2](https://slsa.dev/spec/v1.2/)
- [Microsoft – Exchange Server Update FAQ](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/post-installation-tasks/security-best-practices/exchange-server-update-faq)
- [CISA – Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)
- [FIRST – CVSS v4.0 Specification](https://www.first.org/cvss/v4.0/specification-document)
- [FIRST – EPSS Model](https://www.first.org/epss/how-it-works.html)
- [Kubernetes – Update a Deployment Without Downtime](https://kubernetes.io/docs/tasks/run-application/update-deployment-rolling/)
- [NIST – SP 800-218, Secure Software Development Framework](https://csrc.nist.gov/pubs/sp/800/218/final)
- [Microsoft Learn – Get-ComputerInfo](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-computerinfo)
- [Microsoft Learn – Get-HotFix](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-hotfix)
- [GNU Coreutils – uname](https://www.gnu.org/software/coreutils/manual/html_node/uname-invocation.html)
- [systemd – os-release](https://www.freedesktop.org/software/systemd/man/latest/os-release.html)
- [RPM – rpm manpage](https://rpm.org/docs/6.0.x/man/rpm.8)
- [Debian – dpkg-query](https://manpages.debian.org/stable/dpkg/dpkg-query.1.en.html)
- [Microsoft Learn – Get-FileHash](https://learn.microsoft.com/powershell/module/microsoft.powershell.utility/get-filehash)
- [GNU Coreutils – SHA-2 utilities](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Microsoft Learn – Get-AuthenticodeSignature](https://learn.microsoft.com/powershell/module/microsoft.powershell.security/get-authenticodesignature)
- [GnuPG – gpgv](https://www.gnupg.org/documentation/manuals/gnupg/gpgv.html)
- [Microsoft Learn – Get-Service](https://learn.microsoft.com/powershell/module/microsoft.powershell.management/get-service)
- [Microsoft Learn – Get-NetTCPConnection](https://learn.microsoft.com/powershell/module/nettcpip/get-nettcpconnection)
- [systemd – systemctl](https://www.freedesktop.org/software/systemd/man/latest/systemctl.html)
- [Linux man-pages – ss](https://man7.org/linux/man-pages/man8/ss.8.html)
- [Microsoft Learn – Test-NetConnection](https://learn.microsoft.com/powershell/module/nettcpip/test-netconnection)
- [OpenBSD – nc](https://man.openbsd.org/nc)
- [Microsoft Learn – Get-WinEvent](https://learn.microsoft.com/powershell/module/microsoft.powershell.diagnostics/get-winevent)
- [systemd – journalctl](https://www.freedesktop.org/software/systemd/man/latest/journalctl.html)
