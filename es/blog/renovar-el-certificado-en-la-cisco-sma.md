---
title: "Renovar el certificado en la Cisco SMA"
navTitle: "Certificado SMA"
description: "Los certificados solo pueden instalarse en la Cisco SMA mediante la CLI, y las versiones actuales de AsyncOS validan toda la cadena durante la importación: sin una CA raíz almacenada, esta falla. El artículo muestra las formas de obtener un nuevo par de claves, el método con OpenSSL en detalle, cómo tratar el error RC2-40-CBC de OpenSSL 3 y cómo importar la CA raíz interna al almacén de confianza de la appliance."
date: "2026-08-04"
kategorie: "Cisco ESA / SMA"
timeToRead: "11 min de lectura"
themen:
  - cisco-esa-sma
  - smtp-mailflow
hauptthema: "cisco-esa-sma"
slug: "renovar-el-certificado-en-la-cisco-sma"
translationId: "article-69d93a1e5e081848"
aiPrompt: |
  Du bist mein Assistent für die Zertifikatserneuerung auf einer Cisco SMA (Secure Email and Web Manager). Führe mich Schritt für Schritt durch den Ablauf aus diesem Artikel: 1. Wahl des Wegs zum Schlüsselpaar (OpenSSL-CSR in der eigenen Umgebung, PFX von der CA oder Umweg über eine ESA), 2. CN- und SAN-Liste für meine Hostnamen, 3. je nach Weg CSR-Erzeugung mit OpenSSL oder Konvertierung der PFX-Datei nach PEM inklusive Umgang mit dem Fehler RC2-40-CBC, 4. bei interner CA Import der Root-CA in die Custom-Liste der Appliance, 5. Installation über certconfig in der CLI, 6. Kontrolle. Frage mich zuerst nach den Hostnamen meiner Appliances und der Quarantäneseite, ob die ausstellende CA intern oder öffentlich ist und welche OpenSSL-Version ich installiert habe. Passe alle Befehle an meine Dateinamen an und erinnere mich vor dem Abschluss daran, die certconfig-Session nicht mit Ctrl+C zu beenden und die Änderung mit commit zu aktivieren.
translationOf: cisco-sma-zertifikat-erneuern
translationSourceHash: c99ce64a5e63875b84c7b6f14a7f2fb7e51290fedbdc93d99201cdc97a743508
translationModel: gpt-5.6-terra
translatedAt: 2026-09-01T09:11:33.454Z
translationReview: automatic
url: https://rafaelpfister.ch/es/blog/renovar-el-certificado-en-la-cisco-sma
---

# Renovar el certificado en la Cisco SMA

La Cisco SMA (Security Management Appliance, actualmente comercializada como Cisco Secure Email and Web Manager) gestiona en muchos entornos de correo la cuarentena centralizada de spam y los informes para las Secure Email Gateways. Su certificado HTTPS cubre la GUI de administración y la página de cuarentena, donde los usuarios finales revisan y liberan sus correos retenidos. Cuando caduca, el flujo de correo no se interrumpe. Sin embargo, la caducidad se hace visible de inmediato: cada acceso a la página de cuarentena termina con una advertencia de certificado en el navegador, y precisamente los usuarios a quienes las formaciones de concienciación enseñan a no continuar ante tales advertencias deben ignorarlas.

Durante una renovación en un proyecto de cliente surgieron dos problemas: primero, OpenSSL 3 respondió al archivo PFX de la CA interna con un error críptico sobre `RC2-40-CBC`, y después la appliance rechazó importar el certificado terminado porque no conocía la CA raíz emisora. A continuación se explican ambos obstáculos y su solución.

## Qué hace la SMA de forma distinta a la ESA

En la ESA, todo el ciclo de vida del certificado puede gestionarse mediante la GUI (`Network > Certificates`). La SMA no puede hacerlo: el certificado de servidor se instala exclusivamente mediante la CLI, con el comando `certconfig` en una sesión SSH. La GUI de la SMA solo muestra certificados; únicamente permite gestionar las listas de autoridades de certificación de confianza, como se verá más adelante.

Además, hay otras dos particularidades:

- El diálogo de pegado solo acepta el formato PEM. Un archivo PFX (PKCS#12) debe convertirse antes de instalarse; las versiones actuales de AsyncOS también ofrecen importación directa de PKCS#12, pero antes hay que transferir el archivo a la appliance.
- Las versiones antiguas de AsyncOS (según la Cisco-Technote) no generan por sí mismas ni claves ni CSR, por lo que el par de claves debe crearse fuera; las tres vías posibles se explican más abajo. Las versiones actuales pueden generar directamente en la appliance un certificado autofirmado junto con CSR mediante `certconfig > CERTIFICATE > NEW`. Sin embargo, esto no sirve para un certificado compartido entre varias appliances, porque la clave privada nunca abandona la appliance.

Un único certificado puede utilizarse opcionalmente para todos los servicios (TLS entrante y saliente, acceso de administración HTTPS, LDAPS) o almacenarse por separado para cada servicio. Esto se controla en el diálogo `certconfig`; la cabecera del comando muestra en todo momento la asignación activa (`Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.`). No existe una pantalla de asignación separada como en la ESA, y tampoco puede modificarse mediante la GUI. En la mayoría de entornos, un certificado para todo es la opción pragmática: la lista de nombres ya cubre los FQDN de las appliances, y pares de claves separados multiplican el esfuerzo en cada renovación.

Que el diálogo pregunte por TLS entrante y saliente en una appliance de cuarentena resulta inicialmente desconcertante, ya que la SMA no está en ninguna ruta MX. Sin embargo, utiliza SMTP en ambas direcciones. Inbound (Receiving) es el lado receptor: las ESA entregan mensajes puestos en cuarentena mediante SMTP a la SMA, a la cuarentena centralizada de spam en el puerto 6025 y a las cuarentenas centralizadas de políticas, virus y brotes en el puerto 7025; estas últimas conexiones están cifradas con TLS de fábrica, y la SMA presenta precisamente este certificado. Outbound (Delivery) es el lado emisor: cuando un usuario libera un mensaje de la cuarentena, la SMA lo entrega ella misma de vuelta al flujo de correo mediante sus rutas SMTP; la appliance también envía como correos propios las notificaciones de cuarentena, los informes programados y las alertas. Para la renovación, esto significa que HTTPS es lo crítico en la práctica; con un certificado para todos los servicios, los dos servicios SMTP quedan incluidos automáticamente.

## Definir nombres: CN y SAN

Independientemente de cómo se obtenga el par de claves, primero debe definirse la lista de nombres. El Common Name debe ser el nombre de host con el que los usuarios acceden a la página de cuarentena. La lista SAN debe incluir además los FQDN de las appliances, para que el acceso directo a la GUI de administración también funcione sin advertencias. Para un entorno con dos appliances, la lista de nombres es la siguiente:

| Campo | Valor |
|---|---|
| CN | `spam-quarantine.example.ch` |
| SAN | `spam-quarantine.example.ch` |
| SAN | `sma01.example.ch` |
| SAN | `sma02.example.ch` |

Dos notas al respecto: desde hace tiempo, los navegadores solo evalúan las entradas SAN; el CN por sí solo no basta. Por tanto, el nombre de host de cuarentena también debe figurar como SAN. Los nombres de host cortos sin parte de dominio (por ejemplo, `SMA01`) solo los emite una CA interna; las CA públicas no firman nombres internos.

## Tres vías hacia el nuevo par de claves

Para un certificado que cubra varias appliances y el nombre de host de cuarentena, el par de claves debe crearse fuera de la appliance. Se han consolidado tres vías:

1. Generar la clave y el CSR con OpenSSL dentro del propio entorno. La clave privada se crea donde se necesita y nunca abandona el entorno. Es la vía recomendada; los detalles se explican en la sección siguiente.
2. La CA genera el par de claves y entrega un archivo PFX. Funciona, pero tiene dos inconvenientes: la clave pasa por manos ajenas (por eso la contraseña debe enviarse por un canal separado y no en el mismo correo que el archivo), y según la herramienta de CA puede devolverse un PFX cifrado con RC2, que OpenSSL 3 solo abre con esfuerzo adicional; más sobre ello abajo.
3. El rodeo a través de una ESA, documentado en la Cisco-Technote: allí, en `Network > Certificates`, crear un certificado con el CN de la SMA, descargar el CSR y hacer que la CA lo firme, volver a cargar el certificado firmado en la ESA y exportarlo todo como PFX. También aquí termina siendo necesaria la conversión a PEM.

## Las opciones más importantes de openssl

Como orientación previa, estos son los subcomandos y opciones de `openssl` que aparecen en este artículo, traducidos libremente de la documentación de OpenSSL:

<details class="options-details">
<summary>Resumen de opciones</summary>

| Opción | Significado |
|---|---|
| `req` | Subcomando para solicitudes de certificado (CSR): crear, mostrar, comprobar |
| `-new` | Genera una nueva solicitud |
| `-newkey rsa:2048` | Genera además un nuevo par de claves RSA de 2048 bits |
| `-noenc` | Escribe la clave privada sin cifrar (hasta OpenSSL 3.0: `-nodes`) |
| `-keyout datei` | Archivo de destino para la clave privada |
| `-out datei` | Archivo de destino para la salida, aquí CSR o PEM |
| `-subj text` | Subject de la solicitud en formato `/C=…/O=…/CN=…` |
| `-addext text` | Añade una extensión a la solicitud, aquí la lista SAN |
| `pkcs12` | Subcomando para contenedores PKCS#12 (PFX): crear y extraer |
| `-in datei` | Archivo de entrada |
| `-legacy` | Carga también el proveedor heredado para algoritmos antiguos como RC2 |
| `list` | Subcomando para mostrar las capacidades de la instalación |
| `-providers` | Lista los proveedores cargados |
| `-provider name` | Carga adicionalmente el proveedor indicado para esta ejecución |
| `s_client` | Subcomando: cliente de prueba TLS para conexiones a un servidor |
| `-connect host:port` | Host de destino y puerto de la conexión TLS |
| `-servername name` | Configura Server Name Indication (SNI) en el handshake TLS |
| `x509` | Subcomando para mostrar y procesar certificados |
| `-noout` | Suprime la salida del certificado codificado |
| `-subject` | Muestra el Subject del certificado |
| `-enddate` | Muestra la fecha de caducidad (notAfter) |

</details>

La documentación de OpenSSL incluye las referencias completas como una página de manual independiente por subcomando: `openssl-req(1)`, `openssl-pkcs12(1)`, `openssl-s_client(1)` y `openssl-x509(1)`.

## Iniciar OpenSSL en Windows

Todos los pasos siguientes se realizan con OpenSSL en un sistema dentro del entorno, por ejemplo, un servidor de administración. Basta con la Light Edition de las compilaciones para Windows de Shining Light Productions; el instalador ocupa unos 6 MB y puede verificarse con la lista de sumas de comprobación publicada por slproweb.

El instalador instala todo en `C:\Program Files\OpenSSL-Win64`, y el ejecutable se encuentra en `bin\openssl.exe`. No se añade a la ruta de búsqueda: quien escriba `openssl` en un símbolo del sistema recién abierto recibirá un mensaje de error. Hay tres opciones:

- Abrir la entrada `Win64 OpenSSL Command Prompt` desde el menú Inicio. Inicia `start.bat` desde el directorio de instalación, configura el entorno y muestra la salida de `openssl version -a`. En esa ventana, `openssl` funciona directamente.
- Indicar la ruta completa: `"C:\Program Files\OpenSSL-Win64\bin\openssl.exe" version`.
- Añadir permanentemente `C:\Program Files\OpenSSL-Win64\bin` a la variable de entorno `Path`; después, `openssl` estará disponible en cualquier shell.

Quien ya use Git para Windows no necesita instalación adicional: incluye su propio OpenSSL (`C:\Program Files\Git\mingw64\bin\openssl.exe`), y en Git Bash se encuentra inmediatamente en la ruta de búsqueda. Las versiones actuales de Git incluyen OpenSSL 3.5 con el proveedor heredado activo, por lo que `-legacy` de la sección sobre conversión PFX también funciona allí. Puede comprobarse así:

```bash
openssl list -providers -provider legacy
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `list` | Muestra las capacidades de la instalación de OpenSSL |
| `-providers` | Lista los proveedores cargados con nombre, versión y estado |
| `-provider legacy` | Carga adicionalmente el proveedor `legacy` para esta ejecución; si aparece en la lista, está disponible |

</details>

Sin embargo, Git Bash tiene una particularidad: interpreta los argumentos que comienzan con `/` como rutas y los reescribe. `-subj "/C=CH/O=Example AG/CN=..."` se convierte en `C:/Program Files/Git/C=CH/O=Example AG/CN=...`, y OpenSSL se detiene:

```text
req: subject name is expected to be in the format /type0=value0/type1=value1/type2=...
where characters may be escaped by \. This name is not in that format:
'C:/Program Files/Git/C=CH/O=Example AG/CN=spam-quarantine.example.ch'
```

Un `MSYS_NO_PATHCONV=1` antepuesto desactiva la reescritura para esa ejecución concreta. El problema no se produce en Símbolo del sistema, PowerShell ni en OpenSSL Command Prompt.

## Generar la clave y el CSR con OpenSSL

Una sola llamada genera la clave y el CSR con la lista SAN completa:

```bash
openssl req -new -newkey rsa:2048 -noenc \
  -keyout spam-quarantine.example.ch.key \
  -out spam-quarantine.example.ch.csr \
  -subj "/C=CH/O=Example AG/CN=spam-quarantine.example.ch" \
  -addext "subjectAltName=DNS:spam-quarantine.example.ch,DNS:sma01.example.ch,DNS:sma02.example.ch"
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-new` | Genera una nueva solicitud de certificado (CSR) |
| `-newkey rsa:2048` | Genera además un nuevo par de claves RSA de 2048 bits |
| `-noenc` | Escribe la clave privada sin cifrar en el archivo |
| `-keyout …` | Archivo de destino para la clave privada |
| `-out …` | Archivo de destino para el CSR |
| `-subj …` | Subject con país, organización y Common Name |
| `-addext …` | Añade a la solicitud la extensión SAN con todos los nombres DNS |

</details>

El archivo CSR se envía a la CA; la clave permanece en el servidor. Se recibe de vuelta el certificado firmado junto con el certificado intermedio, normalmente directamente como PEM. Así queda todo preparado para la instalación y la conversión PFX se omite completamente en esta vía.

El archivo de clave no está cifrado (`-noenc`), porque `certconfig` lo espera precisamente así. Hasta la instalación, debe mantenerse en el servidor con permisos restrictivos; después, se elimina o se guarda en el gestor de contraseñas.

## Convertir PFX a PEM

Esta sección y la siguiente se aplican a las vías 2 y 3, que terminan con un archivo PFX. `certconfig` espera el certificado y la clave privada como PEM, con la clave sin cifrar. Una sola llamada de OpenSSL se encarga de ambos:

```bash
openssl pkcs12 -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `pkcs12` | Subcomando para crear y extraer contenedores PKCS#12 |
| `-in …` | El archivo PFX de entrada |
| `-out …` | El archivo PEM de salida con certificado, clave y certificados de cadena |
| `-noenc` | Escribe la clave privada sin frase de contraseña (hasta OpenSSL 3.0 la opción se llamaba `-nodes`) |

</details>

La solicitud de la contraseña de importación se realiza sin eco, y tampoco aparecen asteriscos. El archivo PEM resultante contiene en un único archivo el certificado, la clave y los certificados de cadena incluidos, por lo que debe protegerse adecuadamente: eliminarlo después de la instalación o guardarlo en el gestor de contraseñas.

## Cuando OpenSSL 3 rechaza el archivo PFX

Con archivos PFX antiguos, la conversión en OpenSSL 3.x se interrumpe con este mensaje:

```text
Error outputting keys and certificates
error:0308010C:digital envelope routines:inner_evp_generic_fetch:unsupported:
crypto/evp/evp_fetch.c: Global default library context, Algorithm (RC2-40-CBC : 0)
```

La causa no es un archivo defectuoso, sino una decisión de diseño: OpenSSL 3 ha trasladado algoritmos antiguos como RC2, RC4 y DES a un proveedor heredado separado que no se carga de forma predeterminada. Sin embargo, muchas exportaciones PFX de sistemas Windows y herramientas de CA antiguos cifran precisamente la parte del certificado del contenedor con RC2-40-CBC. OpenSSL 1.1 abría estos archivos sin problemas; OpenSSL 3 los rechaza.

La solución es una única opción adicional:

```bash
openssl pkcs12 -legacy -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `-legacy` | Carga el proveedor heredado para esta ejecución; así vuelven a estar disponibles algoritmos antiguos como RC2-40-CBC y la conversión se completa |

</details>

El requisito es una instalación de OpenSSL que incluya el proveedor heredado; las compilaciones comunes de Windows lo incluyen.

Quien quiera eliminar el error de forma permanente puede actuar en el origen y exportar el archivo PFX con cifrado moderno: los diálogos de exportación y las herramientas de CA actuales ofrecen AES-256, lo que elimina por completo el rodeo del proveedor heredado.

Como alternativa gráfica, funciona XCA (X Certificate and Key Management): importar el archivo PFX mediante `Importieren > PKCS#12`, exportar después el certificado como PEM en la pestaña `Zertifikate` y la clave por separado como PEM sin cifrar en la pestaña `Private Schlüssel`. Se necesitan ambas exportaciones, ya que `certconfig` solicita el certificado y la clave por separado. XCA incluye su propia biblioteca criptográfica y también abre contenedores con algoritmos heredados.

Una nota sobre la fuente de descarga: el propio proyecto OpenSSL no publica binarios para Windows, sino que remite a compilaciones de terceros como Win64 OpenSSL de Shining Light Productions. Los portales de descarga con sus propios instaladores no son la dirección adecuada para una herramienta criptográfica.

## Primero, importar la CA raíz interna en el almacén de confianza de la appliance

Las versiones actuales de AsyncOS validan toda la cadena al crear un perfil de certificado. Si el certificado procede de una CA interna cuya raíz la appliance no conoce, la importación se interrumpe con este mensaje:

```text
Cannot import certificate: The certificate authority validation could not be
trusted for certificate because unable to get local issuer certificate
```

La appliance mantiene dos listas de autoridades de certificación de confianza: la lista de sistema incluida y una lista personalizada para CA propias. La CA raíz interna debe añadirse a la lista personalizada antes de instalar el certificado de servidor. Solo se necesita el certificado público de la CA como archivo PEM (`-----BEGIN CERTIFICATE-----` hasta `-----END CERTIFICATE-----`), no una clave privada.

Así se transfiere la CA raíz a la appliance mediante la interfaz web:

1. Abrir `Network > Certificates`.
2. En la sección `Certificate Authorities`, hacer clic en `Edit Settings`.
3. En `Custom List`, seleccionar la opción `Enable`.
4. Cargar el archivo PEM mediante `Choose File`.
5. Ejecutar `Submit` y, a continuación, `Commit Changes`.
6. En `Network > Certificates > Manage Trusted Root Certificates`, comprobar que la CA aparezca en la lista de certificados personalizados.

Si ya existe una lista personalizada, expórtela primero y añada la nueva CA al paquete PEM existente: la importación reemplaza la lista; de lo contrario, desaparecerán las CA añadidas anteriormente. En una cadena con nivel intermedio, importe primero la CA raíz y después la CA intermedia. AsyncOS comprueba durante la importación, entre otras cosas, la fecha de caducidad, los duplicados y la marca `CA:TRUE`, y rechaza una CA intermedia mientras falte la raíz correspondiente. La misma importación también es posible mediante la CLI: `certconfig > CERTAUTHORITY > CUSTOM > IMPORT`, después `commit`.

Dos delimitaciones: para actualizaciones a través de un proxy que inspecciona TLS, la SMA mantiene un almacén de confianza separado (`updateconfig > TRUSTED_CERTIFICATES > ADD`), donde la lista de CA personalizada no se aplica. Y la CA raíz en la SMA no elimina las advertencias del navegador: los clientes siguen necesitando la raíz mediante su propia distribución de certificados, normalmente por GPO, y la appliance debe entregar el certificado de servidor junto con el certificado intermedio.

## Instalación con certconfig

Inicie sesión en la SMA mediante SSH y ejecute `certconfig`. En las versiones actuales de AsyncOS, el diálogo trabaja con perfiles de certificado:

```text
sma01.example.ch> certconfig

Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.

Choose the operation you want to perform:
- CERTIFICATE - Import, Create a request, Edit or Remove Certificate Profiles
- CERTAUTHORITY - Manage System and Customized Authorities
- CRL - Manage Certificate Revocation Lists
[]> certificate
```

Detrás de `CERTIFICATE` se encuentran las operaciones `IMPORT` (archivo PKCS#12 cargado previamente en la appliance), `PASTE` (pegar el certificado en la CLI), `NEW` (generar un certificado autofirmado junto con CSR), `EDIT`, `EXPORT`, `DELETE` y `PRINT` (muestra la asignación a los servicios). La vía habitual mediante SSH es `PASTE`: el diálogo solicita un nombre para el perfil y, a continuación, el certificado, la clave privada y opcionalmente el certificado intermedio de la CA, cada uno como bloque PEM y terminado con un único `.` en una línea propia. La pregunta final sobre la comprobación FQDN del Common Name puede responderse con el valor predeterminado. El certificado intermedio debe incluirse en el perfil; de lo contrario, faltará la cadena para los clientes y, según el navegador, la advertencia puede persistir pese a que el certificado sea válido.

Las versiones antiguas de AsyncOS (según la Cisco-Technote) muestran en su lugar un diálogo `SETUP`. Comienza con la pregunta `Do you want to use one certificate/key for receiving, delivery, HTTPS management access, and LDAPS?`: un `y` asigna el mismo par a los cuatro servicios; un `n` recorre la consulta de certificado, clave e intermedio una vez por servicio. El principio de pegado es idéntico.

Dos puntos determinan el éxito o el fracaso: no finalice la sesión con Ctrl+C, ya que eso descarta todos los cambios de inmediato. Y ejecute `commit` al final; solo entonces el certificado queda activo. Con dos appliances, el proceso debe repetirse en ambas, ya que la configuración de certificados no se sincroniza entre SMAs.

## Comprobación

La prueba más rápida se realiza desde fuera contra la página de cuarentena. El acceso de usuario final a la cuarentena de spam está en el puerto HTTPS 83 de forma predeterminada, salvo que se haya configurado otra cosa al activarlo:

```bash
openssl s_client -connect spam-quarantine.example.ch:83 \
  -servername spam-quarantine.example.ch </dev/null 2>/dev/null |
  openssl x509 -noout -subject -enddate
```

<details class="options-details">
<summary>Opciones explicadas</summary>

| Opción | Efecto |
|---|---|
| `s_client` | Cliente de prueba TLS: establece la conexión y retransmite el certificado presentado |
| `-connect …:83` | Host y puerto de destino, aquí el puerto HTTPS de la cuarentena de spam |
| `-servername …` | Configura Server Name Indication (SNI) para que el servidor entregue el certificado adecuado |
| `x509` | Procesa el certificado retransmitido |
| `-noout` | Suprime la salida del certificado codificado |
| `-subject` | Muestra el Subject del certificado |
| `-enddate` | Muestra la fecha de caducidad (notAfter) |

</details>

La salida debe mostrar el nuevo Subject y la nueva fecha de caducidad. En la appliance, `certconfig` lista los certificados activos mediante la operación `PRINT`, y la comprobación en el navegador contra la GUI de administración y la página de cuarentena confirma que la cadena se ha construido correctamente.

## Fuentes

1.  [How to Generate and Install a Certificate on an SMA](https://www.cisco.com/c/en/us/support/docs/security/content-security-management-appliance/118460-technote-sma-00.html): Cisco-Technote con el procedimiento certconfig de las versiones antiguas de AsyncOS, el requisito PEM y las vías para generar certificados mediante ESA u OpenSSL.

2.  [Common Administrative Tasks, AsyncOS 16.0 para Cisco Secure Email and Web Manager](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_01011.html): capítulo de la guía de administración sobre la gestión de las listas de Certificate Authority (listas de sistema y personalizada), incluidas las comprobaciones al importar CA.

3.  [Comprehensive Spam Quarantine Setup Guide on ESA and SMA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118692-configure-esa-00.html): guía de Cisco sobre la cuarentena de spam, incluido el acceso de usuario final mediante HTTPS en el puerto 83.

4.  [openssl-req](https://docs.openssl.org/master/man1/openssl-req/): referencia para generar clave y CSR, incluido `-addext` para la lista SAN.

5.  [openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): referencia de las opciones de conversión, entre ellas `-noenc` (anteriormente `-nodes`) y `-legacy`.

6.  [OpenSSL 3.0 Migration Guide](https://docs.openssl.org/3.0/man7/migration_guide/): contexto sobre el traslado de algoritmos antiguos al proveedor heredado.

7.  [XCA: X Certificate and Key Management](https://hohnstaedt.de/xca/): herramienta de código abierto para importar y exportar estructuras PKCS#12 y PEM.

8.  [Win32/Win64 OpenSSL](https://slproweb.com/products/Win32OpenSSL.html): compilaciones para Windows de Shining Light Productions, a las que remite el proyecto OpenSSL, incluida la lista de sumas de comprobación publicada.

9.  [MSYS2: Filesystem Paths](https://www.msys2.org/docs/filesystem-paths/): descripción de la reescritura automática de rutas que modifica en Git Bash el argumento `-subj`, junto con `MSYS_NO_PATHCONV`.

10.  [openssl-s_client](https://docs.openssl.org/master/man1/openssl-s_client/): referencia del cliente de prueba TLS, entre otras cosas `-connect` y `-servername`.

11.  [openssl-x509](https://docs.openssl.org/master/man1/openssl-x509/): referencia de las opciones de visualización, entre ellas `-noout`, `-subject` y `-enddate`.
