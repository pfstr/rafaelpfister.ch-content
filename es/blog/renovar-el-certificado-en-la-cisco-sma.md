---
title: "Renovar el certificado en la Cisco SMA"
navTitle: "Certificado SMA"
description: "Los certificados en la Cisco SMA solo pueden instalarse mediante la CLI, y las versiones actuales de AsyncOS validan toda la cadena durante la importación: sin una CA raíz almacenada, esta falla. El artículo muestra las vías para obtener un nuevo par de claves, el método con OpenSSL en detalle, cómo gestionar el error RC2-40-CBC de OpenSSL 3 y cómo importar la CA raíz interna al almacén de confianza de la appliance."
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
url: https://rafaelpfister.ch/es/blog/renovar-el-certificado-en-la-cisco-sma
translationSourceHash: 0c12510db6a327680d08d3f4eb6924738cef4987860e42c41043ce66467d4249
translationModel: gpt-5.6-terra
translatedAt: 2026-08-05T06:08:47.457Z
translationReview: automatic
---

# Renovar el certificado en la Cisco SMA

La Cisco SMA (Security Management Appliance, ahora denominada Cisco Secure Email and Web Manager) se encarga en muchos entornos de correo de la cuarentena centralizada de spam y de los informes para las Secure Email Gateways. Su certificado HTTPS cubre la GUI de administración y la página de cuarentena, donde los usuarios finales revisan y liberan sus correos retenidos. Si caduca, el flujo de correo no se interrumpe. Sin embargo, la caducidad se hace visible de inmediato: cada acceso a la página de cuarentena termina con una advertencia de certificado en el navegador, y precisamente los usuarios a quienes las formaciones de concienciación enseñan a no continuar ante estas advertencias deben ignorarlas entonces.

Durante una renovación en un proyecto de cliente surgieron dos obstáculos: primero, OpenSSL 3 respondió al archivo PFX de la CA interna con un error críptico sobre `RC2-40-CBC`, y después la appliance rechazó la importación del certificado terminado porque no conocía la CA raíz emisora. Ambas dificultades y sus soluciones se explican más abajo.

## Qué hace la SMA de forma distinta a la ESA

En la ESA, todo el ciclo de vida del certificado puede gestionarse desde la GUI (`Network > Certificates`). La SMA no puede hacerlo: el certificado de servidor se instala exclusivamente mediante la CLI, con el comando `certconfig` en una sesión SSH. La GUI de la SMA solo muestra certificados; allí únicamente se pueden gestionar las listas de autoridades de certificación de confianza, como se verá más adelante.

Además, hay otras dos particularidades:

- El diálogo de pegado solo acepta el formato PEM. Un archivo PFX (PKCS#12) debe convertirse antes de la instalación; las versiones actuales de AsyncOS también ofrecen una importación directa de PKCS#12, pero para ello el archivo debe llegar primero a la appliance.
- Las versiones antiguas de AsyncOS (las cubiertas por la nota técnica de Cisco) no generan por sí mismas ni claves ni CSR; el par de claves debe crearse fuera de la appliance. A continuación se explican las tres vías posibles. Las versiones actuales pueden generar directamente en la appliance un certificado autofirmado junto con su CSR mediante `certconfig > CERTIFICATE > NEW`. Sin embargo, esto no ayuda para un certificado común para varias appliances, porque la clave privada nunca sale de la appliance.

Un único certificado puede servir opcionalmente para todos los servicios (TLS entrante y saliente, acceso de administración HTTPS, LDAPS) o configurarse por separado para cada servicio. Esto se controla en el diálogo `certconfig`; la cabecera del comando muestra en todo momento la asignación activa (`Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.`). No existe una pantalla de asignación separada como en la ESA, y tampoco se puede modificar desde la GUI. En la mayoría de los entornos, un certificado para todo es la opción pragmática: la lista de nombres ya cubre los FQDN de las appliances, y los pares de claves separados multiplican el esfuerzo en cada renovación.

Que el diálogo de una appliance de cuarentena pregunte por TLS entrante y saliente resulta confuso a primera vista, pues la SMA no se encuentra en ninguna ruta MX. Aun así, utiliza SMTP en ambos sentidos. Inbound (Receiving) es el lado de recepción: las ESA entregan mensajes en cuarentena mediante SMTP a la SMA, a la cuarentena central de spam en el puerto 6025 y a las cuarentenas centrales de políticas, virus y brotes en el puerto 7025; estas últimas conexiones están cifradas con TLS de fábrica, y la SMA presenta precisamente este certificado. Outbound (Delivery) es el lado de envío: cuando un usuario libera un mensaje de la cuarentena, la SMA lo entrega ella misma de vuelta al flujo de correo mediante sus rutas SMTP, y la appliance también envía como correos propios las notificaciones de cuarentena, los informes programados y las alertas. Para la renovación, esto significa que HTTPS es lo crítico en la práctica; ambos servicios SMTP quedan cubiertos simplemente al usar un certificado para todos los servicios.

## Definir nombres: CN y SAN

Independientemente de la vía elegida para el par de claves, primero hay que definir la lista de nombres. El Common Name debe ser el nombre de host con el que los usuarios acceden a la página de cuarentena. La lista SAN debe incluir además los FQDN de las appliances, para que el acceso directo a la GUI de administración también funcione sin advertencias. Para un entorno con dos appliances, la lista de nombres es la siguiente:

| Campo | Valor |
|---|---|
| CN | `spam-quarantine.example.ch` |
| SAN | `spam-quarantine.example.ch` |
| SAN | `sma01.example.ch` |
| SAN | `sma02.example.ch` |

Dos notas al respecto: los navegadores hace tiempo que solo evalúan las entradas SAN; el CN por sí solo no basta. Por tanto, el nombre de host de la cuarentena también debe figurar como SAN. Y los nombres de host cortos sin parte de dominio (por ejemplo, `SMA01`) solo los emite una CA interna; las CA públicas no firman nombres internos.

## Tres vías para obtener el nuevo par de claves

Para un certificado que cubra varias appliances y el nombre de host de cuarentena, el par de claves debe generarse fuera de la appliance. Se han consolidado tres vías:

1. Generar la clave y el CSR con OpenSSL dentro del propio entorno. La clave privada se crea donde se necesita y nunca abandona el entorno. Es la vía recomendada; los detalles se explican en la siguiente sección.
2. La CA genera el par de claves y entrega un archivo PFX. Funciona, pero tiene dos inconvenientes: la clave pasa por manos ajenas (por eso la contraseña debe enviarse por un canal separado y no en el mismo correo que el archivo), y según la herramienta de CA, puede recibirse un PFX cifrado con RC2 que OpenSSL 3 solo abre con trabajo adicional; más información abajo.
3. El desvío a través de una ESA, documentado en la nota técnica de Cisco: allí, en `Network > Certificates`, crear un certificado con el CN de la SMA, descargar el CSR y hacer que la CA lo firme, volver a cargar el certificado firmado en la ESA y exportarlo todo como PFX. También aquí se requiere al final la conversión a PEM.

## Iniciar OpenSSL en Windows

Todos los pasos siguientes se realizan con OpenSSL, en un sistema dentro del entorno, por ejemplo un servidor de administración. La edición Light de las compilaciones para Windows de Shining Light Productions es suficiente; el instalador ocupa unos 6 MB y puede verificarse contra la lista de sumas de comprobación publicada por slproweb.

El instalador coloca todo bajo `C:\Program Files\OpenSSL-Win64`, y el ejecutable se encuentra en `bin\openssl.exe`. No se añade a la ruta de búsqueda: quien escriba `openssl` en un símbolo del sistema nuevo recibirá un mensaje de error. Hay tres formas de resolverlo:

- Abrir la entrada `Win64 OpenSSL Command Prompt` desde el menú Inicio. Inicia `start.bat` desde el directorio de instalación, configura el entorno y muestra la salida de `openssl version -a`. En esta ventana, `openssl` funciona directamente.
- Indicar la ruta completa: `"C:\Program Files\OpenSSL-Win64\bin\openssl.exe" version`.
- Añadir `C:\Program Files\OpenSSL-Win64\bin` de forma permanente a la variable de entorno `Path`; después, `openssl` estará disponible en cualquier shell.

Quien ya utilice Git para Windows no necesita instalar nada adicional: incluye su propio OpenSSL (`C:\Program Files\Git\mingw64\bin\openssl.exe`), y en Git Bash está inmediatamente en la ruta de búsqueda. Las versiones actuales de Git incluyen OpenSSL 3.5 con el proveedor Legacy activo, por lo que `-legacy` de la sección sobre conversión PFX también funciona allí. Se puede comprobar así:

```bash
openssl list -providers -provider legacy
```

Sin embargo, Git Bash tiene un inconveniente: interpreta como rutas los argumentos que comienzan por `/` y los reescribe. `-subj "/C=CH/O=Example AG/CN=..."` se convierte en `C:/Program Files/Git/C=CH/O=Example AG/CN=...`, y OpenSSL se detiene:

```text
req: subject name is expected to be in the format /type0=value0/type1=value1/type2=...
where characters may be escaped by \. This name is not in that format:
'C:/Program Files/Git/C=CH/O=Example AG/CN=spam-quarantine.example.ch'
```

Anteponer `MSYS_NO_PATHCONV=1` desactiva la reescritura para esa llamada concreta. El problema no se produce en el símbolo del sistema, PowerShell ni en el OpenSSL Command Prompt.

## Generar la clave y el CSR con OpenSSL

Una sola llamada genera la clave y el CSR con la lista SAN completa:

```bash
openssl req -new -newkey rsa:2048 -noenc \
  -keyout spam-quarantine.example.ch.key \
  -out spam-quarantine.example.ch.csr \
  -subj "/C=CH/O=Example AG/CN=spam-quarantine.example.ch" \
  -addext "subjectAltName=DNS:spam-quarantine.example.ch,DNS:sma01.example.ch,DNS:sma02.example.ch"
```

El archivo CSR se envía a la CA; la clave permanece en el servidor. De vuelta se recibe el certificado firmado junto con el intermedio, normalmente directamente como PEM. Con ello está todo listo para la instalación; por esta vía se omite por completo la conversión de PFX.

El archivo de clave no está cifrado (`-noenc`), porque `certconfig` lo espera exactamente así. Hasta la instalación, debe mantenerse en el servidor con permisos restrictivos; después se elimina o se guarda en el gestor de contraseñas.

## Convertir PFX a PEM

Esta sección y la siguiente se aplican a las vías 2 y 3, que terminan con un archivo PFX. `certconfig` espera el certificado y la clave privada como PEM, con la clave sin cifrar. Una sola llamada de OpenSSL realiza ambas tareas:

```bash
openssl pkcs12 -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

`-noenc` (hasta OpenSSL 3.0, la opción se llamaba `-nodes`) escribe la clave privada sin frase de contraseña en el archivo de salida. La solicitud de la contraseña de importación se realiza sin eco y tampoco aparecen asteriscos. El archivo PEM resultante contiene el certificado, la clave y los certificados de cadena incluidos en un solo archivo, por lo que debe protegerse adecuadamente: eliminarlo tras la instalación o guardarlo en el gestor de contraseñas.

## Cuando OpenSSL 3 rechaza el archivo PFX

Con archivos PFX antiguos, la conversión se interrumpe en OpenSSL 3.x con este mensaje:

```text
Error outputting keys and certificates
error:0308010C:digital envelope routines:inner_evp_generic_fetch:unsupported:
crypto/evp/evp_fetch.c: Global default library context, Algorithm (RC2-40-CBC : 0)
```

La causa no es un archivo defectuoso, sino una decisión de diseño: OpenSSL 3 ha trasladado algoritmos antiguos como RC2, RC4 y DES a un proveedor Legacy separado, que no se carga de forma predeterminada. Sin embargo, muchas exportaciones PFX de sistemas Windows y herramientas de CA antiguos cifran la parte de certificado del contenedor precisamente con RC2-40-CBC. OpenSSL 1.1 abría estos archivos sin problemas; OpenSSL 3 los rechaza.

La solución es una única opción adicional:

```bash
openssl pkcs12 -legacy -in spam-quarantine.example.ch.pfx -out spam-quarantine.example.ch.pem -noenc
```

`-legacy` carga además el proveedor Legacy para esta llamada, y entonces la conversión se completa. El requisito es una instalación de OpenSSL que incluya el proveedor Legacy; las compilaciones habituales para Windows lo incorporan.

Quien quiera eliminar el error de forma permanente debe actuar en el origen y exportar el archivo PFX con cifrado moderno: los diálogos de exportación y las herramientas de CA actuales ofrecen AES-256, con lo que el desvío mediante Legacy deja de ser necesario.

Como alternativa gráfica, funciona XCA (X Certificate and Key Management): importar el archivo PFX mediante `Importieren > PKCS#12`, exportar después el certificado como PEM en la pestaña `Zertifikate` y la clave por separado como PEM sin cifrar en la pestaña `Private Schlüssel`. Se necesitan ambas exportaciones; `certconfig` solicita el certificado y la clave por separado. XCA incluye su propia biblioteca criptográfica y también abre contenedores con algoritmos Legacy.

Una nota más sobre la fuente de descarga: el proyecto OpenSSL no publica binarios para Windows por sí mismo, sino que remite a compilaciones de terceros como Win64 OpenSSL de Shining Light Productions. Los portales de descarga con instaladores propios no son la dirección adecuada para una herramienta criptográfica.

## Importar primero la CA raíz interna al almacén de confianza de la appliance

Las versiones actuales de AsyncOS validan toda la cadena al crear un perfil de certificado. Si el certificado procede de una CA interna cuya raíz la appliance no conoce, la importación se interrumpe con este mensaje:

```text
Cannot import certificate: The certificate authority validation could not be
trusted for certificate because unable to get local issuer certificate
```

La appliance mantiene dos listas de autoridades de certificación de confianza: la lista del sistema incluida y una lista personalizada para CA propias. La CA raíz interna debe añadirse a la lista personalizada antes de instalar el certificado de servidor. Solo se necesita el certificado público de la CA como archivo PEM (`-----BEGIN CERTIFICATE-----` hasta `-----END CERTIFICATE-----`), no una clave privada.

Así se carga la CA raíz en la appliance mediante la interfaz web:

1. Abrir `Network > Certificates`.
2. En la sección `Certificate Authorities`, hacer clic en `Edit Settings`.
3. En `Custom List`, seleccionar la opción `Enable`.
4. Cargar el archivo PEM mediante `Choose File`.
5. Ejecutar `Submit` y después `Commit Changes`.
6. En `Network > Certificates > Manage Trusted Root Certificates`, comprobar que la CA aparezca en la lista de certificados personalizados.

Si ya existe una lista personalizada, expórtela primero y añada la nueva CA al paquete PEM existente: la importación reemplaza la lista, por lo que de otro modo desaparecerían las CA configuradas anteriormente. En una cadena con una autoridad intermedia, importe primero la CA raíz y después la CA intermedia. Durante la importación, AsyncOS comprueba, entre otras cosas, la fecha de caducidad, los duplicados y el indicador `CA:TRUE` configurado, y rechaza una intermedia mientras falte la raíz correspondiente. La misma importación también puede realizarse mediante la CLI: `certconfig > CERTAUTHORITY > CUSTOM > IMPORT`, después `commit`.

Dos precisiones: para actualizaciones mediante un proxy que inspecciona TLS, la SMA utiliza un almacén de confianza separado (`updateconfig > TRUSTED_CERTIFICATES > ADD`), al que no se aplica la lista de CA personalizadas. Y añadir la CA raíz a la SMA no elimina las advertencias del navegador: los clientes siguen necesitando la raíz mediante su propia distribución de certificados, normalmente a través de GPO, y la appliance debe entregar el certificado de servidor junto con el intermedio.

## Instalación con certconfig

Inicie sesión en la SMA por SSH y ejecute `certconfig`. En las versiones actuales de AsyncOS, el diálogo trabaja con perfiles de certificado:

```text
sma01.example.ch> certconfig

Currently using one certificate/key for receiving, delivery, HTTPS management access, and LDAPS.

Choose the operation you want to perform:
- CERTIFICATE - Import, Create a request, Edit or Remove Certificate Profiles
- CERTAUTHORITY - Manage System and Customized Authorities
- CRL - Manage Certificate Revocation Lists
[]> certificate
```

Detrás de `CERTIFICATE` se encuentran las operaciones `IMPORT` (archivo PKCS#12 cargado previamente en la appliance), `PASTE` (pegar certificado en la CLI), `NEW` (generar certificado autofirmado junto con CSR), `EDIT`, `EXPORT`, `DELETE` y `PRINT` (muestra la asignación a los servicios). La vía habitual por SSH es `PASTE`: el diálogo solicita un nombre para el perfil, seguido del certificado, la clave privada y opcionalmente el certificado intermedio de la CA, cada uno como bloque PEM y terminado con un único `.` en una línea propia. La pregunta final sobre la comprobación del FQDN del Common Name puede responderse con el valor predeterminado. El intermedio debe incluirse en el perfil; de lo contrario, a los clientes les faltará la cadena y, según el navegador, la advertencia puede persistir pese a un certificado válido.

Las versiones antiguas de AsyncOS (las cubiertas por la nota técnica de Cisco) muestran en su lugar un diálogo `SETUP`. Comienza con la pregunta `Do you want to use one certificate/key for receiving, delivery, HTTPS management access, and LDAPS?`: un `y` asigna el mismo par a los cuatro servicios, mientras que un `n` recorre la consulta de certificado, clave e intermedio una vez por servicio. El principio de pegado es idéntico.

Dos puntos determinan el éxito o el fracaso: no termine la sesión con Ctrl+C, pues esto descarta inmediatamente todos los cambios. Y al final ejecute `commit`; solo entonces el certificado estará activo. Con dos appliances, el procedimiento se repite en ambas; la configuración de certificados no se sincroniza entre las SMA.

## Comprobación

La prueba más rápida se realiza desde fuera contra la página de cuarentena. El acceso de los usuarios finales a la cuarentena de spam utiliza de forma predeterminada el puerto HTTPS 83, siempre que no se haya configurado otro al activarlo:

```bash
openssl s_client -connect spam-quarantine.example.ch:83 \
  -servername spam-quarantine.example.ch </dev/null 2>/dev/null |
  openssl x509 -noout -subject -enddate
```

La salida debe mostrar el nuevo Subject y la nueva fecha de caducidad. En la appliance, `certconfig` con la operación `PRINT` muestra los certificados activos, y la comprobación en el navegador contra la GUI de administración y la página de cuarentena confirma que la cadena se ha construido correctamente.

## Fuentes

1.  [How to Generate and Install a Certificate on an SMA](https://www.cisco.com/c/en/us/support/docs/security/content-security-management-appliance/118460-technote-sma-00.html): Nota técnica de Cisco con el procedimiento certconfig de versiones antiguas de AsyncOS, el requisito de PEM y las vías para generar certificados mediante ESA u OpenSSL.

2.  [Common Administrative Tasks, AsyncOS 16.0 for Cisco Secure Email and Web Manager](https://www.cisco.com/c/en/us/td/docs/security/security_management/sma/sma16-0/user_guide/b_sma_admin_guide_16_0/b_NGSMA_Admin_Guide_chapter_01011.html): Capítulo de la guía de administración sobre la gestión de listas de Certificate Authority (listas del sistema y personalizadas), incluidas las comprobaciones durante la importación de CA.

3.  [Comprehensive Spam Quarantine Setup Guide on ESA and SMA](https://www.cisco.com/c/en/us/support/docs/security/email-security-appliance/118692-configure-esa-00.html): Guía de Cisco para la cuarentena de spam, incluido el acceso de usuarios finales mediante HTTPS en el puerto 83.

4.  [openssl-req](https://docs.openssl.org/master/man1/openssl-req/): Referencia para generar clave y CSR, incluido `-addext` para la lista SAN.

5.  [openssl-pkcs12](https://docs.openssl.org/master/man1/openssl-pkcs12/): Referencia de las opciones de conversión, entre ellas `-noenc` (anteriormente `-nodes`) y `-legacy`.

6.  [OpenSSL 3.0 Migration Guide](https://docs.openssl.org/3.0/man7/migration_guide/): Contexto sobre el traslado de algoritmos antiguos al proveedor Legacy.

7.  [XCA: X Certificate and Key Management](https://hohnstaedt.de/xca/): Herramienta de código abierto para importar y exportar estructuras PKCS#12 y PEM.

8.  [Win32/Win64 OpenSSL](https://slproweb.com/products/Win32OpenSSL.html): Compilaciones para Windows de Shining Light Productions, a las que remite el proyecto OpenSSL, incluida la lista de sumas de comprobación publicada.

9.  [MSYS2: Filesystem Paths](https://www.msys2.org/docs/filesystem-paths/): Descripción de la reescritura automática de rutas que modifica el argumento `-subj` en Git Bash, junto con `MSYS_NO_PATHCONV`.
