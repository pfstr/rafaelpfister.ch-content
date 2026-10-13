---
title: "Proton Drive: modelo de datos criptográfico y funcionamiento de la sincronización"
blatt: "proton-drive"
description: "Proton Drive para administradores de almacenamiento, seguridad y plataformas: almacenamiento centralizado en la nube con cifrado de extremo a extremo del lado del cliente, volúmenes, recursos compartidos, nodos, revisiones y bloques, jerarquía de claves y firmas OpenPGP, Drive SDK, sincronización de escritorio y archivos bajo demanda, automatización CLI, compartición, conflictos, versiones, registro, recuperación e historia técnica."
fakten:
  - label: Función del sistema
    wert: almacenamiento centralizado de archivos en la nube con cifrado de extremo a extremo del lado del cliente, sincronización, compartición y versionado
    href: https://proton.me/support/drive
  - label: Modelo de confianza
    wert: El servidor almacena contenidos y nombres cifrados; el texto sin cifrar y el material de claves desbloqueado se generan en el cliente autorizado
    href: https://proton.me/blog/protondrive-security
  - label: Modelo de datos
    wert: Un volumen contiene recursos compartidos; un recurso compartido apunta a una raíz de árbol; los archivos y las carpetas son nodos con enlaces, claves y revisiones
    href: https://proton.me/blog/protondrive-security
  - label: Transferencia de archivos
    wert: Las revisiones constan de bloques cifrados; el cliente cifra, transfiere, verifica y vuelve a ensamblar el objeto
    href: https://proton.me/blog/drive-cryptography-update
  - label: Criptografía
    wert: jerarquía de claves basada en OpenPGP; los archivos nuevos utilizan v6 PKESK, v2 SEIPD y AES-GCM, mientras que las revisiones antiguas permanecen vinculadas a su esquema
    href: https://proton.me/blog/drive-cryptography-update
  - label: Metadatos
    wert: El contenido y los nombres de archivos y carpetas son E2EE; según Proton, determinados campos como el tamaño del archivo siguen siendo legibles por el servidor
    href: https://proton.me/support/drive
  - label: Integridad
    wert: la información firmada de claves, árboles y bloques, así como las firmas OpenPGP, detectan manipulaciones
    href: https://proton.me/support/drive-signature-management
  - label: Núcleo del cliente
    wert: Drive SDK unifica las operaciones de archivos y la criptografía entre clientes web, de escritorio, móviles y CLI
    href: https://github.com/ProtonDriveApps/sdk
  - label: Sincronización de escritorio
    wert: Windows y macOS integran marcadores de posición bajo demanda, estados locales sin conexión y sincronización en segundo plano en Explorer o Finder
    href: https://proton.me/support/proton-drive-windows-on-demand-sync
  - label: CLI
    wert: programa oficial independiente para Windows, macOS y Linux; operaciones de archivos y compartición, pero sin motor permanente de sincronización de escritorio
    href: https://proton.me/support/drive-cli
  - label: Núcleo de recuperación
    wert: El acceso a la cuenta por sí solo no basta después de restablecer la contraseña; la clave anterior se desbloquea de nuevo con la contraseña previa, Recovery Phrase o Recovery File
    href: https://proton.me/support/drive-data-recovery
  - label: Regla de copia de seguridad
    wert: La sincronización y el historial de versiones no son una copia de seguridad independiente; los datos críticos requieren exportación separada, sumas de comprobación, retención y prueba de restauración
    href: https://csrc.nist.gov/pubs/sp/800/34/r1/final
werbung:
  - newsletter
ctaThemen:
  - proton-drive
translationSourceHash: 6063c7d85190de63a8973bc7cdb304fff10f81fd617a2044cf40f8371849717c
translationModel: gpt-5.6-terra
translatedAt: 2026-10-11T10:48:33.566Z
translationReview: automatic
---

# Proton Drive: modelo de datos criptográfico y funcionamiento de la sincronización

Proton Drive es un almacenamiento centralizado de archivos en la nube cuyos clientes cifran los contenidos y los metadatos esenciales antes de cargarlos. El servicio combina acceso web, sincronización de escritorio, aplicaciones móviles, recursos compartidos, versiones y una línea de comandos con un modelo de datos cifrado de extremo a extremo. La [vista general de soporte](https://proton.me/support/drive) indica acceso web, Windows, macOS, Android e iOS/iPadOS; la [Drive CLI](https://proton.me/support/drive-cli) oficial añade operaciones automatizables en Windows, macOS y Linux.

Para los administradores, Proton Drive no es ni un recurso compartido de archivos clásico ni un simple portal de carga. Un cliente autorizado gestiona la sesión de la cuenta y las claves, resuelve un árbol cifrado, observa los cambios locales, genera revisiones, cifra bloques y sincroniza los eventos del servidor de vuelta al sistema de archivos. Los servidores siguen siendo responsables de la disponibilidad, la autorización, la cuota de almacenamiento, las versiones y los recursos compartidos, sin poder procesar el contenido de los archivos ni los nombres cifrados en texto sin cifrar.

La explicación sigue un archivo desde el cliente, pasando por las claves, los bloques cifrados y la API, hasta el almacenamiento del lado del servidor. La sincronización, los recursos compartidos, el versionado, el diagnóstico y la copia de seguridad independiente se basan en esta ruta de datos.

Proton Drive almacena los datos de forma centralizada, pero traslada el cifrado y la verificación de firmas a clientes de confianza. El artículo sigue un archivo desde el cambio local mediante la jerarquía de claves, los bloques y la revisión hasta la sincronización, la compartición y la recuperación.

## Arquitectura: servicio centralizado, cliente de confianza

Proton eligió un almacenamiento centralizado en la nube en lugar de una red entre pares. Los servidores proporcionan API, almacenamiento de objetos, eventos, membresías, cuotas, versiones y disponibilidad; los clientes se encargan del procesamiento de texto sin cifrar y de la criptografía. Proton explica la justificación de este modelo en la [clasificación del almacenamiento centralizado y descentralizado](https://proton.me/blog/centalized-vs-p2p-protondrive).

La ruta técnica tiene seis niveles:

1. **Identidad y sesión:** El cliente autentica una cuenta de Proton, carga claves de dirección cifradas y las desbloquea localmente.
2. **Volumen y recurso compartido:** Un volumen delimita el espacio de almacenamiento. Los recursos compartidos definen raíces de árbol, miembros y permisos.
3. **Nodo y enlace:** Un archivo o carpeta es un nodo; el enlace lo sitúa en la jerarquía cifrada.
4. **Revisión y bloque:** Un archivo puede tener varias revisiones. El contenido se divide en bloques y se cifra del lado del cliente.
5. **API y almacenamiento de objetos:** El cliente transmite metadatos cifrados, paquetes de claves, manifiestos y bloques mediante HTTPS.
6. **Sistema de archivos local:** Los clientes de escritorio proyectan el árbol en la nube como marcador de posición o archivo local, observan cambios y resuelven conflictos.

El [modelo de seguridad de Proton Drive](https://proton.me/blog/protondrive-security) documenta volumen, recurso compartido, nodo, enlace y claves jerárquicas. La [actualización criptográfica de 2026](https://proton.me/blog/drive-cryptography-update) describe además nodo, revisión, bloque y paquete de claves de contenido del motor de cliente compartido.

<iframe class="kb-infographic" style="aspect-ratio: 1400 / 1076" src="/images/kb-interaktiv-proton-drive.svg?v=20260813" title="Interaktive Infografik: Proton Drive von Konto und Share über Node-, Revisions- und Blockverschlüsselung bis zu API, SDK, Desktop-Sync, CLI, Sharing, Versionierung, Recovery und unabhängiger Sicherung" loading="lazy">
  <a href="/images/kb-interaktiv-proton-drive.svg?v=20260813">Abrir gráfico interactivo directamente</a>.
</iframe>

## Modelo de datos: volumen, recurso compartido, nodo, enlace y revisión

Un **volumen** es un espacio de almacenamiento asignado. Su recurso compartido predeterminado apunta a la raíz del árbol de directorios privado. Otros **recursos compartidos** pueden apuntar a archivos o carpetas y vincular miembros con funciones. Por tanto, un recurso compartido es a la vez un punto de acceso y un límite criptográfico.

Un **nodo** describe un archivo o carpeta y contiene atributos técnicos y material criptográfico. Un **enlace** posiciona el nodo en el árbol y lo conecta con su nodo padre. Para los archivos, las **revisiones** gestionan estados consecutivos de contenido; una revisión hace referencia a **bloques** cifrados. La separación conceptual explica fenómenos operativos importantes:

- Cambiar el nombre modifica la designación cifrada en el árbol, pero no necesariamente el contenido del archivo.
- Mover modifica la relación con el padre y, con ello, la información de enlace protegida criptográficamente.
- Reemplazar genera una revisión del mismo objeto de archivo, mientras que «conservar ambos» genera un segundo nodo.
- Compartir delega el acceso a una raíz de recurso compartido, en lugar de descifrar en el servidor todos los contenidos que se encuentran debajo.
- Eliminar mueve primero una entrada a la papelera de Proton Drive; las papeleras locales representan un estado independiente.

La [gestión oficial de versiones](https://proton.me/support/version-history) describe Replace, Keep both, Skip y la restauración de revisiones anteriores. La [documentación sobre eliminación y restauración](https://proton.me/support/proton-drive-delete-restore-synced-files) explica la diferencia entre la papelera en la nube y la papelera local.

El volumen, el recurso compartido, el nodo y la revisión describen qué objeto se almacena. La jerarquía de claves explica a continuación quién puede descifrar y firmar nombres, metadatos y contenido.

## Jerarquía de claves y OpenPGP

Las claves y frases de contraseña se generan en el cliente y solo se transmiten cifradas al servidor. El modelo de seguridad emplea varios niveles:

1. Una dirección de Proton posee un par de claves asimétricas.
2. Para un recurso compartido se generan una frase de contraseña aleatoria y una clave de recurso compartido.
3. La frase de contraseña del recurso compartido se cifra y firma para cada miembro autorizado con su clave de dirección.
4. Cada nodo de archivo o carpeta posee claves de nodo y una frase de contraseña de nodo.
5. La frase de contraseña del nodo queda protegida con la clave de la carpeta padre o de la raíz del recurso compartido.
6. El nombre y otras cargas útiles se cifran en este contexto jerárquico.
7. Los contenidos de archivos usan una Content Key simétrica, cuyo paquete de claves está protegido por la clave de nodo.

OpenPGP sirve como formato de mensajes y paquetes de claves. La criptografía asimétrica protege las claves de sesión simétricas; el cifrado simétrico procesa eficientemente los datos propiamente dichos. La base normativa de OpenPGP es [RFC 9580](https://datatracker.ietf.org/doc/html/rfc9580).

Para los archivos creados recientemente desde la transición criptográfica, Proton describe paquetes v6-PKESK, v2-SEIPD y AES-GCM. El esquema de un archivo permanece vinculado a sus revisiones: al reemplazarlo, un cliente debe usar el esquema de paquetes correspondiente al objeto de archivo existente, o la revisión se rechaza ([actualización criptográfica](https://proton.me/blog/drive-cryptography-update)). Este es un motivo de interoperabilidad para actualizar los clientes de forma controlada.

## Cifrado, firmado y legible por el servidor

«Cifrado de extremo a extremo» no significa que el servidor no vea ningún dato operativo. Proton menciona en la [vista general de soporte de Drive](https://proton.me/support/drive) el contenido, los nombres de archivos y los nombres de carpetas como E2EE, pero también indica que determinados campos, como el tamaño del archivo, siguen siendo legibles por el servidor para permitir funciones como la clasificación.

Para un modelo de amenazas, las clases de datos se separan:

| Clase de datos | Ruta de protección | Relevancia administrativa |
|---|---|---|
| Contenido del archivo | bloques cifrados del lado del cliente | El servidor no puede indexar ni reparar el contenido |
| Nombre de archivo/carpeta | carga útil cifrada en el árbol | La búsqueda de nombres en el servidor es limitada |
| Paquetes de claves | cifrados y firmados | La recuperación depende de las claves privadas |
| Firmas/manifiestos | prueba de integridad y autoría | Las advertencias no deben ignorarse de forma generalizada |
| Tamaño del archivo | campo legible por el servidor | Los patrones de volumen y tráfico permanecen visibles |
| Relación con la cuenta, sesión y recurso compartido | metadatos del plano de control | La autorización y la auditoría siguen estando del lado del servidor |
| IP, hora, versión del cliente | metadatos operativos/de seguridad | Parte del modelo de amenazas del proveedor y del endpoint |

Las firmas digitales confirman el cargador y las cargas útiles sin modificar. La [gestión de firmas](https://proton.me/support/drive-signature-management) explica la verificación de firmas OpenPGP y las advertencias. Una firma no demuestra que el contenido sea inocuo ni técnicamente correcto; acredita la procedencia o la inalterabilidad en el contexto de claves verificado.

## Bloques, revisiones y límite de confirmación

Los archivos se dividen en bloques en el cliente. El cliente genera o adopta la Content Key, cifra los bloques, los transfiere y publica la revisión con la información criptográfica correspondiente. El servidor puede almacenar y entregar bloques sin conocer el texto sin cifrar.

El progreso de carga aún no es una confirmación funcional. Un administrador distingue:

- archivo local leído por completo,
- todos los bloques cifrados,
- todos los bloques transferidos correctamente,
- revisión aceptada por el servidor,
- cursor de eventos avanzado,
- archivo resoluble en un segundo cliente,
- firma y contenido verificados durante la descarga.

En descargas grandes, el cliente descifra y verifica los datos localmente. Para grandes volúmenes de datos, el cliente web utiliza una ruta de streaming con Service Worker, de modo que no sea necesario mantener todo el archivo en la memoria del navegador; Proton describe este proceso en [descarga web y streaming](https://proton.me/blog/proton-drive-web-encryption-technique).

## Drive SDK y estructura del cliente

El ecosistema de clientes surgió originalmente de implementaciones específicas de cada plataforma. El **Drive SDK** compartido unifica las rutas de cuenta, API, caché, nodo, recurso compartido, carga, descarga y criptografía. El código fuente está en el [repositorio oficial de Proton Drive SDK](https://github.com/ProtonDriveApps/sdk); el contrato del paquete JavaScript/TypeScript se publica como [`@protontech/drive-sdk`](https://www.npmjs.com/package/@protontech/drive-sdk).

La organización oficial de GitHub muestra las principales pilas tecnológicas ([ProtonDriveApps](https://github.com/ProtonDriveApps)):

| Cliente/componente | Pila principal | Función operativa |
|---|---|---|
| Web | TypeScript/JavaScript y criptografía del navegador | acceso interactivo, streaming, compartición |
| Drive SDK / CLI | TypeScript; CLI como programa único empaquetado con Bun | operaciones de archivos compartidas y automatización |
| [Windows](https://github.com/ProtonDriveApps/windows-drive) | C#/.NET más componentes SDK/Crypto compartidos | sincronización en segundo plano, Explorer y Cloud Files |
| [macOS](https://github.com/ProtonDriveApps/mac-drive) | Swift más componentes SDK compartidos | sincronización en segundo plano, Finder/File Provider |
| Android | Kotlin | archivos móviles, estado sin conexión y fotos |
| iOS/iPadOS | Swift | archivos móviles, estado sin conexión y fotos |

El enfoque SDK reduce las diferencias entre las implementaciones de criptografía y archivos, pero no elimina los límites de las plataformas. Las reglas de nombres de Windows, el comportamiento de macOS File Provider, los límites de ejecución en segundo plano de los dispositivos móviles y la integración de escritorio de Linux siguen siendo tareas de cada cliente.

La API criptográfica aún no es un servicio de sincronización de archivos. El cliente de escritorio observa los cambios locales y remotos, los relaciona y decide si cargar, descargar o resolver un conflicto.

## Sincronización de escritorio como máquina de estados

La sincronización de escritorio no mantiene una simple copia de bytes, sino que reconcilia dos espacios de nombres cambiantes. Un ciclo abstracto es:

1. Cargar el estado del sistema de archivos local y el cursor de eventos de la nube.
2. Determinar cambios locales, eventos del servidor y transferencias pendientes.
3. Normalizar nombres y reglas de plataforma o marcar conflictos.
4. Resolver la jerarquía de claves y los nodos necesarios.
5. Planificar carga, descarga, movimiento, eliminación o hidratación.
6. Transferir el contenido en bloques y confirmar la revisión.
7. Actualizar de forma atómica los marcadores de posición, atributos, caché y cursor locales.
8. Reintentar errores con una referencia estable al objeto o escalarlos de forma visible.

El estado de sincronización «sincronizado» solo es significativo para el cliente y cursor considerados. Un segundo cliente, un recurso compartido o una copia de seguridad independiente proporcionan una prueba de extremo a extremo más sólida.

## Archivos bajo demanda y estados locales

Windows y macOS muestran el árbol en la nube en el administrador de archivos sin almacenar todos los contenidos permanentemente de forma local. La [documentación de Windows sobre archivos bajo demanda](https://proton.me/support/proton-drive-windows-on-demand-sync) distingue **Available when online**, **Available on this device** y **Always available on this device**. La [guía de macOS](https://proton.me/support/drive-macos-guide) contempla Online Only, disponibilidad local temporal y Make Available Offline.

Por tanto, un nombre de archivo visible no demuestra que exista una copia local. Para copias de seguridad, antivirus, indexadores, sistemas de compilación o flujos de trabajo de adjuntos, debe estar claro si el archivo:

- existe solo como marcador de posición,
- se hidrata al abrirse,
- se guarda temporalmente en caché local,
- se ha fijado explícitamente para uso sin conexión,
- se ha modificado localmente, pero todavía no se ha cargado,
- solo está en este dispositivo debido a un error.

Anidar dos clientes de nube bajo demanda en el mismo árbol es arriesgado. La [ayuda de errores de Windows](https://proton.me/support/proton-drive-windows-common-issues) menciona expresamente los archivos de otro almacenamiento en la nube como un problema cuando solo están disponibles en línea. Una tarea de copia de seguridad fija localmente la cantidad de entrada o utiliza la CLI oficial para transferencias explícitas.

## Conflictos, nombres y funcionamiento multidispositivo

Los conflictos surgen cuando el cliente y la nube modifican de manera independiente el mismo nodo lógico o cuando la plataforma de destino no puede representar un nombre. Las clases típicas son:

- cambio simultáneo de contenido en la misma revisión,
- eliminación frente a modificación,
- cambio de nombre frente a movimiento,
- mayúsculas/minúsculas en plataformas con distinta semántica de mayúsculas y minúsculas,
- caracteres no permitidos o nombres reservados en Windows,
- cargas con el mismo nombre mediante Replace, Keep both o Skip,
- cambio sin conexión tras un retraso prolongado de eventos,
- archivo local bloqueado por una aplicación durante la transferencia.

Proton documenta que un conflicto puede generar un archivo renombrado o una nueva versión ([Shared with me en Windows](https://proton.me/support/shared-with-me-windows)). La ayuda de errores de Windows describe archivos abiertos y nombres no permitidos ([problemas frecuentes de sincronización](https://proton.me/support/proton-drive-windows-common-issues)). Los administradores no buscan solo «errores», sino también copias en conflicto, cargas pendientes y archivos que nunca llegaron a la nube debido a reglas de nombres.

## Inventario de clientes, binarios y registros

Un endpoint administrado inventaría la plataforma, el paquete de cliente, la firma, el inicio automático, la cuenta, la raíz de sincronización, la disponibilidad local, el espacio libre y la ruta de registros. Los números de versión pertenecen al inventario de tiempo de ejecución y no a un dato estático del artículo.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Proton-Drive-Clientinventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-Process ProtonDrive -ErrorAction SilentlyContinue
Get-AuthenticodeSignature "$env:LOCALAPPDATA\Proton\Drive\ProtonDrive.exe"
Get-ChildItem "$env:LOCALAPPDATA\Proton\ProtonDrive\Logs"
proton-drive version</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">proton-drive version
file ./proton-drive
sha256sum ./proton-drive
ldd ./proton-drive</code></pre>
  </div>
</div>

[`Get-Process`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process) comprueba el proceso de Windows; [`Get-AuthenticodeSignature`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature) la firma de código; [`Get-ChildItem`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem) inventaría los registros. En Unix, [`file`](https://man7.org/linux/man-pages/man1/file.1.html) identifica el binario, [`sha256sum`](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html) su hash y [`ldd`](https://man7.org/linux/man-pages/man1/ldd.1.html) las dependencias dinámicas. El acceso compatible a registros para Windows y macOS figura en [Proton Drive Logs](https://proton.me/support/proton-drive-logs).

## CLI: operación explícita en lugar de sincronización en segundo plano

La CLI de Proton Drive es un cliente basado en SDK para Windows, macOS y Linux. El inicio de sesión se realiza mediante un flujo de navegador; según la [introducción a la CLI](https://proton.me/blog/proton-drive-cli), las sesiones se almacenan en Windows Credential Manager, macOS Keychain o libsecret, respectivamente. La CLI lista, carga y descarga archivos, y gestiona archivos, papelera, recursos compartidos e invitaciones. Se ejecuta como una operación individual y finaliza después.

El límite operativo más importante está documentado explícitamente: la CLI no sustituye un motor de sincronización de escritorio. Una carga programada no detecta automáticamente eliminaciones, cambios de nombre, conflictos bidireccionales ni el éxito en un segundo dispositivo. Quien use la CLI como transporte de copias de seguridad debe implementar por sí mismo el inventario de origen, la dirección, la regla de eliminación, la repetición, el código de salida, la verificación del destino y la retención.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Anmeldung und Proton-Drive-Inventar">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">proton-drive auth login
proton-drive filesystem list /my-files
proton-drive filesystem list /my-files --json | ConvertFrom-Json
proton-drive sharing status /my-files/Reports</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">proton-drive auth login
proton-drive filesystem list /my-files
proton-drive filesystem list /my-files --json | jq .
proton-drive sharing status /my-files/Reports</code></pre>
  </div>
</div>

Todos los comandos y ayudas de la CLI se documentan en [Using Proton Drive CLI](https://proton.me/support/drive-cli). [`ConvertFrom-Json`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json) y [`jq`](https://jqlang.org/manual/) procesan la salida legible por máquinas. Las automatizaciones se basan en campos JSON y en el código de salida, no en textos de pantalla localizados.

## Carga, estrategia de conflictos y verificación

Una carga repetible especifica el origen local, la raíz de destino y la estrategia de conflictos. Los comodines los expande el shell que realiza la llamada y pueden proporcionar conjuntos distintos en Windows y Unix. Por ello, las tareas críticas generan previamente un manifiesto con ruta, tamaño y hash.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Proton-Drive-Upload und Manifest">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Get-ChildItem .\reports -File -Recurse |
  Get-FileHash -Algorithm SHA256 |
  Export-Csv .\reports-manifest.csv -NoTypeInformation
proton-drive filesystem upload .\reports\* /my-files/Reports --conflict-strategy skip
proton-drive filesystem list /my-files/Reports --json | Out-File .\drive-result.json</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">find ./reports -type f -print0 | sort -z | xargs -0 sha256sum > reports-manifest.sha256
proton-drive filesystem upload ./reports/* /my-files/Reports --conflict-strategy skip
proton-drive filesystem list /my-files/Reports --json > drive-result.json
jq . drive-result.json</code></pre>
  </div>
</div>

[`Get-FileHash`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash), [`Export-Csv`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv), [`find`](https://man7.org/linux/man-pages/man1/find.1.html), [`sort`](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html) y [`xargs`](https://man7.org/linux/man-pages/man1/xargs.1.html) generan el conjunto local esperado. Sin embargo, la lista del proveedor aún no demuestra la igualdad de contenido. Para datos críticos, sigue una descarga a un destino separado y una comparación de hashes.

## Descarga y aceptación de extremo a extremo

Una prueba de descarga utiliza una nueva ruta de destino para que ningún archivo local existente oculte el resultado. A continuación, se comprueba frente al manifiesto de origen. Si hay varias revisiones, la prueba debe determinar qué revisión se espera.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Proton-Drive-Downloadprüfung">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">New-Item -ItemType Directory .\restore-test
proton-drive filesystem download /my-files/Reports .\restore-test
Get-ChildItem .\restore-test -File -Recurse |
  Get-FileHash -Algorithm SHA256 |
  Export-Csv .\restore-manifest.csv -NoTypeInformation</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">mkdir -p ./restore-test
proton-drive filesystem download /my-files/Reports ./restore-test
find ./restore-test -type f -print0 | sort -z | xargs -0 sha256sum > restore-manifest.sha256
diff -u reports-manifest.sha256 restore-manifest.sha256</code></pre>
  </div>
</div>

[`New-Item`](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/new-item) y [`mkdir`](https://www.gnu.org/software/coreutils/manual/html_node/mkdir-invocation.html) crean el destino aislado; [`diff`](https://www.gnu.org/software/diffutils/manual/html_node/Invoking-diff.html) compara los manifiestos. Las rutas absolutas o con prefijos diferentes se normalizan antes de la comparación.

## Compartición con cuentas y mediante enlaces públicos

Proton Drive admite al menos dos modelos de compartición:

- **Membresía/invitación:** Una persona se convierte en miembro de un recurso compartido con función de Viewer o Editor. La frase de contraseña del recurso compartido se empaqueta para la clave de dirección del miembro.
- **Enlace público:** Un enlace permite el acceso sin una cuenta de Proton. Una contraseña protege la frase de contraseña del recurso compartido; el enlace y la contraseña constituyen conjuntamente el secreto de acceso.

La [gestión de acceso](https://proton.me/support/drive-manage-access-shared-files) documenta funciones, delegación de Editor, revocación y configuraciones separadas para invitación por correo electrónico y Public Link. [Contraseña y fecha de vencimiento](https://proton.me/support/password-protect-files-proton-drive) limitan el acceso mediante enlaces. Si se envía una contraseña aleatoria como fragmento de URL, el navegador y los destinatarios ven el enlace completo; aunque el fragmento no se envía al servidor como ruta HTTP, puede filtrarse por el portapapeles, el historial del navegador, capturas de pantalla o telemetría del endpoint.

Los administradores inventarían propietario, raíz del recurso compartido, miembros, función, derecho de reenvío, estado del enlace público, fecha de vencimiento y última revisión. Según la configuración, un Editor puede compartir por sí mismo y modificar permisos. Por ello, el offboarding no solo revoca la cuenta, sino que revisa todos los recursos compartidos y enlaces externos.

## Versiones, papelera y copia de seguridad

El historial de versiones almacena revisiones anteriores del mismo objeto de archivo. Una revisión restaurada no elimina automáticamente las versiones más recientes. La papelera conserva los nodos eliminados hasta la eliminación definitiva. Ambas funciones son valiosas, pero siguen estando dentro del mismo ámbito administrativo y criptográfico de fallos.

No protegen de forma fiable contra:

- pérdida o imposibilidad de recuperación de la clave de la cuenta,
- permisos incorrectos o eliminación definitiva maliciosa,
- eventos de cuota, contrato y proveedor,
- errores que surgieron antes de la primera revisión exitosa en la nube,
- omisiones de sincronización no detectadas,
- compromiso de identidad o endpoint a nivel de toda la organización.

Una copia de seguridad independiente lee una cantidad definida de datos de nube con un cliente autorizado, la escribe en un destino administrado por separado, la dota de manifiesto, retención e historial inmutable y realiza pruebas de restauración. NIST SP 800-34 trata la copia de seguridad, el almacenamiento alternativo y las pruebas de recuperación como controles de planificación de contingencia interrelacionados ([NIST SP 800-34 Rev. 1](https://csrc.nist.gov/pubs/sp/800/34/r1/final)). Los fundamentos se encuentran en [Backup and Disaster Recovery](/kb/backup-dr) y [Storage](/kb/storage).

## Identidad, sesión y confianza del endpoint

El cifrado de extremo a extremo desplaza la confianza del proveedor al cliente, la cuenta y el endpoint. Quien controle el dispositivo desbloqueado, la sesión del navegador o la cuenta puede descifrar y firmar cambios en nombre del usuario. Por ello, MFA, llaves de seguridad de hardware, protección del dispositivo, revocación de sesiones, procedencia del cliente y protección de la caché local forman un conjunto.

El [modelo de amenazas](https://proton.me/blog/proton-drive-threat-model) menciona, entre otros, dispositivos comprometidos, phishing y la autenticidad de los clientes oficiales como límites. La disponibilidad del código fuente permite revisarlo, pero no sustituye la verificación del binario que realmente se ejecuta ni de su ruta de actualización.

La detección de malware también es limitada: la [detección opcional de archivos maliciosos](https://proton.me/support/proton-drive-malware-protection) compara opcionalmente un hash con hashes conocidos de malware. No descifra todo el inventario en el servidor, no detecta contenidos desconocidos y no sustituye un sensor de endpoint.

## Restablecimiento de contraseña y recuperación de datos

La recuperación de la cuenta y la recuperación de datos son procesos distintos. Un restablecimiento de contraseña puede recuperar el acceso a la cuenta, pero inicialmente puede hacer inaccesibles las claves privadas anteriores. Proton Drive muestra entonces los datos antiguos como bloqueados.

La [recuperación de datos de Drive](https://proton.me/support/drive-data-recovery) oficial indica la contraseña anterior, Recovery Phrase o Recovery File como vías de desbloqueo. Las nuevas cargas tras el restablecimiento pueden generarse con claves nuevas, mientras que los datos antiguos permanecen bloqueados; las fotos y los álbumes pueden requerir un paso adicional de fusión.

Por ello, un runbook de organización documenta:

- quién puede iniciar un restablecimiento de contraseña,
- qué métodos de recuperación se han configurado y probado previamente,
- dónde se guardan sin conexión y con protección de acceso la Recovery Phrase o el Recovery File,
- cómo se reconocen los estados de claves antiguos y nuevos,
- cómo se revisan los recursos compartidos y miembros tras la recuperación,
- cuándo no se debe eliminar en ningún caso una unidad bloqueada.

«Unlock later» conserva la posibilidad de una recuperación posterior. «Delete locked drive» es una eliminación irreversible de datos y no debe formar parte de un procedimiento estándar no confirmado.

Si la identidad, las claves y el estado local son plausibles, sigue la ruta de red. El proxy, TLS, los endpoints de API y las transferencias bloqueadas deben comprobarse por separado del modelo criptográfico de datos.

## Ruta de red y diagnóstico del transporte

HTTPS protege el transporte entre el cliente y la API de Proton o los endpoints de almacenamiento. No sustituye el cifrado de extremo a extremo, pero protege los metadatos de sesión y operación, la identidad del endpoint y la disponibilidad. Proxy, filtros DNS, inspección TLS, VPN, MTU o conexiones inestables pueden afectar de forma distinta a las solicitudes del plano de control y a las transferencias de bloques.

<div class="os-tabs" data-os-tabs>
  <div class="os-tabs-bar" role="tablist" aria-label="Betriebssystem für Proton-Drive-Netzwerkdiagnose">
    <button type="button" role="tab" aria-selected="true" data-os-tab="windows">Windows</button>
    <button type="button" role="tab" aria-selected="false" data-os-tab="unix">Linux &amp; Unix</button>
  </div>
  <div role="tabpanel" data-os-panel="windows">
    <pre><code class="language-powershell">Resolve-DnsName drive.proton.me
Test-NetConnection drive.proton.me -Port 443
curl.exe -sS -D - -o NUL https://drive.proton.me/
Get-NetTCPConnection -State Established | Where-Object RemotePort -eq 443</code></pre>
  </div>
  <div role="tabpanel" data-os-panel="unix" hidden>
    <pre><code class="language-bash">getent ahosts drive.proton.me
curl -sS -D - -o /dev/null https://drive.proton.me/
ss -ntp state established '( dport = :443 )'
openssl s_client -connect drive.proton.me:443 -servername drive.proton.me -brief</code></pre>
  </div>
</div>

[`Resolve-DnsName`](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname) y [`getent`](https://man7.org/linux/man-pages/man1/getent.1.html) comprueban la resolución de nombres; [`Test-NetConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection) TCP 443; [`curl`](https://curl.se/docs/manpage.html) HTTP y redireccionamientos; [`Get-NetTCPConnection`](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection) o [`ss`](https://man7.org/linux/man-pages/man8/ss.8.html) conexiones activas; [`openssl s_client`](https://docs.openssl.org/master/man1/openssl-s_client/) TLS y SNI. La prueba contra el dominio web no demuestra que todos los endpoints de API/bloques sean accesibles; los registros del cliente proporcionan la clase de destino y solicitud que realmente falló.

## Observabilidad y cadena de diagnóstico

Un diagnóstico útil comienza por el objeto funcional:

1. Determinar cuenta, recurso compartido, ruta lógica y cliente afectado.
2. Distinguir el estado local: marcador de posición, hidratado, fijado, modificado, bloqueado o erróneo.
3. Comprobar la vista de actividad y los registros en busca de referencias a nodo, revisión o transferencia.
4. Controlar el espacio local libre y la cuota de nube.
5. Comprobar la compatibilidad de nombres y rutas.
6. Comprobar DNS, TCP, TLS y proxy/VPN con el endpoint real.
7. Tratar las advertencias de firma, descifrado y claves por separado de los errores de transporte.
8. Listar o descargar el archivo con la CLI oficial o un segundo cliente.
9. Comparar el hash de descarga y la revisión esperada.
10. Solo entonces considerar pausa/reanudación, reautenticación, actualización o intervención en la caché.

Los [errores de sincronización de Windows](https://proton.me/support/proton-drive-windows-common-issues) mencionan progreso, archivos abiertos, otros clientes de nube, pausa/reanudación, reglas de nombres y red. Estas ayudas de usuario se complementan en la operación administrativa con inventario de objetos y verificación de extremo a extremo. La metodología general se encuentra en [Troubleshooting](/kb/troubleshooting).

## Modelo de rendimiento y capacidad

Los costes no se deben solo a los bytes transferidos. Muchos archivos pequeños generan trabajo de claves, metadatos, API y firmas por nodo. Los árboles profundos requieren una resolución repetida de jerarquías. En cambio, el contenido de un archivo grande puede procesarse de forma más eficiente en bloques paralelos. Proton indica en la ayuda de errores de Windows que muchos archivos pequeños o profundamente anidados pueden sincronizarse más lentamente que un único archivo del mismo tamaño.

La planificación de capacidad tiene en cuenta:

- cuota en la nube, incluidas versiones y papelera,
- estados locales sin conexión y de caché,
- espacio temporal para carga, descarga y copias en conflicto,
- CPU para criptografía y hashing,
- número de archivos y profundidad del árbol,
- retraso de eventos tras periodos sin conexión,
- comportamiento de uso razonable/limitación de velocidad en transferencias automatizadas.

La introducción de la CLI advierte de no volver a cargar por completo datos sin cambios de forma repetida y remite al uso razonable ([Proton Drive CLI](https://proton.me/blog/proton-drive-cli)). Por ello, una tarea programada mantiene su propio manifiesto de origen o comprueba el estado de destino, en lugar de tratar cada ejecución como una carga completa.

## Funcionamiento con Windows, macOS y Linux

Windows y macOS ofrecen integración con el administrador de archivos y sincronización en segundo plano. Las aplicaciones móviles gestionan carga, descarga, objetos sin conexión y fotos, pero no son un sustituto idéntico de un servicio de sistema de archivos de escritorio. La CLI oficial permite automatizar operaciones de archivos y compartición en los tres sistemas operativos de escritorio.

En junio de 2026, Proton publicó la CLI y describió una aplicación de escritorio Linux completa con sincronización en segundo plano como un proyecto separado aún en desarrollo ([anuncio de la CLI](https://proton.me/blog/proton-drive-cli)). Esta delimitación histórica sigue siendo importante: incluso si la gama de plataformas evoluciona, **operación CLI**, **montaje** y **motor de sincronización bidireccional** no se equiparan. Para cada despliegue se consulta la [página de notas de versión](https://proton.me/support/drive-release-notes) como fuente dependiente del tiempo.

En tareas de servidor, la autenticación mediante navegador se ejecuta de forma controlada y el token de sesión resultante se guarda en el almacén de secretos del sistema operativo de una cuenta de servicio dedicada. Una tarea sin interfaz gráfica no recibe el directorio personal de un administrador. Se conservan el código de salida, el resultado JSON, el manifiesto de origen y la verificación de destino; los secretos y enlaces compartidos se eliminan de los registros.

Las versiones y la papelera protegen frente a errores operativos individuales, pero no sustituyen una recuperación independiente. El runbook debe tratar conjuntamente la cuenta, el acceso a las claves, los datos locales y la exportación verificada.

## Runbook de copia de seguridad y recuperación

Proton Drive puede ser el destino de una copia de seguridad, pero no sustituye un diseño de recuperación. Un runbook sólido incluye:

1. Definir la cantidad de origen y la fuente de verdad.
2. Hidratar completamente los datos locales bajo demanda antes de una copia de seguridad basada en archivos o exportarlos directamente mediante CLI.
3. Generar un manifiesto de archivos con ruta, tamaño y hash.
4. Ejecutar la carga con una regla explícita de conflicto y eliminación.
5. Archivar el código de salida de la CLI y la lista del destino.
6. Realizar una muestra o descarga completa en una ruta aislada.
7. Comparar hashes y revisiones esperadas.
8. Documentar recursos compartidos, versionado, papelera y retención.
9. Probar por separado la recuperación de la cuenta de Proton y de los datos.
10. Mantener una segunda copia independiente fuera del mismo dominio de cuenta y proveedor.

El RPO se refiere a la última revisión confirmada y verificada en Proton, no al momento de escritura del archivo local. El RTO incluye acceso a la cuenta, desbloqueo de claves, listado, descarga, descifrado, ancho de banda y restauración del destino.

## Historia técnica

Proton desarrolló Drive como complemento del ecosistema Proton cifrado. El modelo de seguridad publicado en 2020 estableció el almacenamiento centralizado con una jerarquía de claves OpenPGP del lado del cliente, nombres cifrados, árboles de nodos, recursos compartidos, bloques firmados y enlaces compartidos. Los clientes web y móviles llegaron antes que los clientes de sincronización de escritorio; el historial de versiones se introdujo en 2023 como una ruta de revisión independiente ([introducción del historial de versiones](https://proton.me/blog/drive-version-history)).

Windows y macOS añadieron sincronización en segundo plano y archivos bajo demanda con integración específica de cada plataforma en Explorer/Finder. Las implementaciones separadas condujeron gradualmente al Drive SDK compartido. A principios de 2026, Proton describió el despliegue del SDK en web, Windows, macOS, Android e iOS ([desarrollo del SDK](https://proton.me/blog/drive-sdk-january-2026)).

En junio de 2026 se produjeron dos avances técnicos: una transición criptográfica común a v6 PKESK, v2 SEIPD y AES-GCM para nuevos objetos de archivo, así como la CLI basada en SDK para Windows, macOS y Linux. Con ello se habilitó un acceso oficialmente automatizable sin tener que reconstruir la lógica criptográfica del cliente. Sin embargo, la arquitectura sigue centrada en el cliente: los servidores almacenan y coordinan; los clientes autorizados descifran, firman y reconcilian.

## Lista de comprobación para administradores

En el almacenamiento cifrado del lado del cliente, deben entenderse conjuntamente el modelo de datos, las claves, la sincronización y la copia de seguridad independiente. La lista de comprobación condensa estos puntos para la autorización operativa.

- **Modelo de datos:** Distinguir volumen, recurso compartido, nodo, enlace, revisión y bloque.
- **Criptografía:** Documentar el contexto de claves de dirección, recurso compartido, nodo y contenido, así como el esquema de paquetes.
- **Metadatos:** Separar los campos E2EE y los campos operativos legibles por el servidor en el modelo de amenazas.
- **Clientes:** Inventariar plataforma, tecnología, SDK, integración del sistema de archivos, firma binaria y ruta de registros.
- **Sincronización:** Supervisar el estado local, el cursor de eventos, las copias en conflicto, los nombres y las transferencias pendientes.
- **CLI:** No tratar una operación explícita de archivos como un motor permanente de sincronización o copia de seguridad.
- **Compartición:** Revisar propietario, miembros, funciones, delegación de Editor, Public Link, contraseña y vencimiento.
- **Recuperación:** Probar por separado la contraseña de cuenta, el desbloqueo de datos, Recovery Phrase/File y el acceso al recurso compartido.
- **Copia de seguridad:** Mantener una copia independiente, manifiesto, versionado y descarga de restauración.
- **Aceptación:** Descifrar el archivo en un segundo cliente o descarga aislada y abrirlo tras verificar firma y hash.

## Fuentes

- [NIST SP 800-34 Rev. 1 – Contingency Planning Guide](https://csrc.nist.gov/pubs/sp/800/34/r1/final)
- [Proton Drive – Vista general de soporte](https://proton.me/support/drive)
- [Proton Drive – CLI](https://proton.me/support/drive-cli)
- [Proton – almacenamiento centralizado y descentralizado](https://proton.me/blog/centalized-vs-p2p-protondrive)
- [Proton Drive – Modelo de seguridad](https://proton.me/blog/protondrive-security)
- [Proton Drive – Actualización criptográfica](https://proton.me/blog/drive-cryptography-update)
- [Proton Drive – Gestión de versiones](https://proton.me/support/version-history)
- [Proton Drive – Eliminar y restaurar archivos sincronizados](https://proton.me/support/proton-drive-delete-restore-synced-files)
- [RFC 9580 – OpenPGP](https://datatracker.ietf.org/doc/html/rfc9580)
- [Proton Drive – Gestión de firmas](https://proton.me/support/drive-signature-management)
- [Proton Drive – Descarga web y streaming](https://proton.me/blog/proton-drive-web-encryption-technique)
- [Proton Drive – Repositorio SDK](https://github.com/ProtonDriveApps/sdk)
- [npm – @protontech/drive-sdk](https://www.npmjs.com/package/@protontech/drive-sdk)
- [GitHub – ProtonDriveApps](https://github.com/ProtonDriveApps)
- [Windows](https://github.com/ProtonDriveApps/windows-drive)
- [macOS](https://github.com/ProtonDriveApps/mac-drive)
- [Proton Drive – Sincronización bajo demanda de Windows](https://proton.me/support/proton-drive-windows-on-demand-sync)
- [Proton Drive – Guía de macOS](https://proton.me/support/drive-macos-guide)
- [Proton Drive – Problemas frecuentes de sincronización de Windows](https://proton.me/support/proton-drive-windows-common-issues)
- [Proton Drive – Shared with me en Windows](https://proton.me/support/shared-with-me-windows)
- [Microsoft – Get-Process](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-process)
- [Microsoft – Get-AuthenticodeSignature](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.security/get-authenticodesignature)
- [Microsoft – Get-ChildItem](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/get-childitem)
- [Linux man-pages – file(1)](https://man7.org/linux/man-pages/man1/file.1.html)
- [GNU Coreutils – utilidades sha2](https://www.gnu.org/software/coreutils/manual/html_node/sha2-utilities.html)
- [Linux man-pages – ldd(1)](https://man7.org/linux/man-pages/man1/ldd.1.html)
- [Proton Drive – Registros](https://proton.me/support/proton-drive-logs)
- [Proton – Introducción a Drive CLI](https://proton.me/blog/proton-drive-cli)
- [Microsoft – ConvertFrom-Json](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-json)
- [jq – Manual](https://jqlang.org/manual/)
- [Microsoft – Get-FileHash](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/get-filehash)
- [Microsoft – Export-Csv](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/export-csv)
- [Linux man-pages – find(1)](https://man7.org/linux/man-pages/man1/find.1.html)
- [GNU Coreutils – sort](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html)
- [Linux man-pages – xargs(1)](https://man7.org/linux/man-pages/man1/xargs.1.html)
- [Microsoft – New-Item](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.management/new-item)
- [GNU Coreutils – mkdir](https://www.gnu.org/software/coreutils/manual/html_node/mkdir-invocation.html)
- [GNU Diffutils – diff](https://www.gnu.org/software/diffutils/manual/html_node/Invoking-diff.html)
- [Proton Drive – Gestión de acceso](https://proton.me/support/drive-manage-access-shared-files)
- [Proton Drive – Protección por contraseña y fecha de vencimiento](https://proton.me/support/password-protect-files-proton-drive)
- [Proton Drive – Modelo de amenazas](https://proton.me/blog/proton-drive-threat-model)
- [Proton Drive – Detección de archivos maliciosos](https://proton.me/support/proton-drive-malware-protection)
- [Proton Drive – Recuperación de datos](https://proton.me/support/drive-data-recovery)
- [Microsoft – Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)
- [Linux man-pages – getent(1)](https://man7.org/linux/man-pages/man1/getent.1.html)
- [Microsoft – Test-NetConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/test-netconnection)
- [curl – Manual](https://curl.se/docs/manpage.html)
- [Microsoft – Get-NetTCPConnection](https://learn.microsoft.com/en-us/powershell/module/nettcpip/get-nettcpconnection)
- [Linux man-pages – ss(8)](https://man7.org/linux/man-pages/man8/ss.8.html)
- [OpenSSL – s_client](https://docs.openssl.org/master/man1/openssl-s_client/)
- [Proton Drive – Notas de versión](https://proton.me/support/drive-release-notes)
- [Proton Drive – Introducción del historial de versiones](https://proton.me/blog/drive-version-history)
- [Proton Drive – Desarrollo del SDK 2026](https://proton.me/blog/drive-sdk-january-2026)
