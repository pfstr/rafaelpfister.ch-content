---
title: "Ejecutar Paperless-ngx con poco espacio de almacenamiento: externalizar documentos a un servicio en la nube"
navTitle: "Paperless con servicio en la nube"
description: "Paperless-ngx solo necesita localmente la base de datos, el índice de búsqueda y las miniaturas; los propios documentos pueden almacenarse en un servicio en la nube. Qué reveló la prueba práctica y cómo realizar la configuración con la plantilla lista en tres comandos."
date: "2026-07-26"
kategorie: "Paperless-ngx"
timeToRead: "6 min de lectura"
themen:
  - paperless-ngx
related:
  - rclone-mount-in-docker-container
  - proton-drive-linux-status
  - cloud-mount-testen-dummy-pdfs
slug: "paperless-ngx-poco-espacio-documentos-cloud"
translationOf: "paperless-dokumente-clouddienst-auslagern"
url: "https://rafaelpfister.ch/es/blog/paperless-ngx-poco-espacio-documentos-cloud"
translationId: article-2f00e7c17fc45664
translationModel: gpt-5.6-terra
translatedAt: 2026-07-28T21:22:44.392Z
translationReview: automatic
translationSourceHash: 81212f097221ec6213025dc5de54f583369799181f72747549102e2b4246e021
---

Paperless-ngx guarda sus documentos en un directorio local, y ese directorio crece con cada escaneo. Sin embargo, Paperless apenas necesita los archivos en el día a día: la búsqueda consulta la base de datos, la lista muestra miniaturas y el archivo propiamente dicho solo se lee al abrirlo. Por eso probé si era posible trasladar el almacenamiento a un servicio en la nube. La herramienta para ello es Rclone, con la que los usuarios de Plex llevan años integrando colecciones multimedia completas desde la nube.

El resultado: **funciona en ambos sentidos**, y la configuración se ha reducido entretanto a tres comandos. Este artículo resume lo que reveló la prueba y cómo puedes configurar tú mismo el sistema. Los detalles técnicos se explican en artículos específicos enlazados al final: propagación de montajes de Docker, trampas de AppArmor, autenticación de dos factores y metodología de medición.

## El principio: el almacenamiento activo permanece local, el almacenamiento frío está en la nube

| Componente | Ubicación | Por qué |
|---|---|---|
| Base de datos (contiene el texto OCR) | local | necesita bloqueo real |
| Índice de búsqueda, miniaturas | local | accesos constantes |
| **Archivos de documentos** | **nube** | se leen rara vez |
| Caché (documentos abiertos recientemente) | local, limitado | los accesos repetidos siguen siendo rápidos |

En Paperless, precisamente el nombre del directorio induce a error: `archive/` **no es almacenamiento frío**, sino que contiene la versión PDF/A que se entrega en cada visualización. Pese a su nombre, pertenece al almacenamiento activo. Los originales que rara vez se necesitan en `originals/` son el verdadero almacenamiento frío. Si quieres ahorrar al máximo, desactiva por completo la copia de archivo con `PAPERLESS_ARCHIVE_FILE_GENERATION=never`; la búsqueda de texto completo no se ve afectada, porque el texto está en la base de datos.

Por cierto, Paperless-ngx no incluye una conexión propia a la nube, ni para S3 ni para `django-storages`. Actualmente, un montaje de sistema de archivos mediante Rclone es la única vía, y funciona con cualquiera de los más de 70 servicios compatibles con Rclone. Proton Drive fue mi elección para la prueba por el cifrado de extremo a extremo; un almacenamiento compatible con S3 es la alternativa más robusta.

## Qué reveló la prueba

Probado con una instancia aislada de Paperless, 40 PDF de prueba generados (13,9 MB) y una cuenta de Proton dedicada:

| Operación | Resultado |
|---|---|
| Abrir un documento por primera vez (desde la nube) | ~1,8 s |
| Abrir de nuevo el mismo documento (desde la caché) | ~20 ms |
| Incorporar un documento nuevo hasta que esté en la nube | ~20 s |
| Lista de documentos, búsqueda de texto completo | 39 ms / 272 ms, funciona también **sin** conexión a la nube |
| Comprobación de integridad (suma de comprobación de cada archivo) | superada, sin discrepancias |
| Fallo del montaje | autorrecuperación sin reiniciar Paperless, verificada |

Así, el espacio de almacenamiento local queda desacoplado del tamaño del archivo: la colección puede crecer, el disco no.

## Cómo configurarlo

La configuración completa está disponible como plantilla en GitHub: [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage). En el servidor:

```bash
git clone https://github.com/pfstr/paperless-cloud-storage.git
cd paperless-cloud-storage
sudo ./setup.sh      # una sola vez, prepara el host (único paso con permisos de root)
./wizard.sh          # guiado: elegir proveedor, credenciales, prueba de ciclo completo
```

El asistente consulta el servicio en la nube (Proton, S3, Backblaze B2, WebDAV, SFTP o «No está en la lista» para cualquier otro servicio de Rclone), comprueba la conexión con una prueba real de subida y bajada e inicia el contenedor de almacenamiento. Después:

- **Instalación nueva:** `docker compose -f paperless.yml up -d`, listo.
- **Instancia existente de Paperless:** la base de datos y la configuración permanecen intactas; la guía [RETROFIT.md](https://github.com/pfstr/paperless-cloud-storage/blob/main/docs/RETROFIT.md) describe la subida de los documentos existentes y el cambio necesario en tu archivo Compose.

He renunciado deliberadamente a una interfaz web. Al principio se utilizó la interfaz web de Rclone, pero los túneles SSH, CORS y los montajes efímeros la hicieron peor que la línea de comandos que debía sustituir. Tres preguntas en la terminal son más rápidas.

## Para que el montaje se mantenga estable en el día a día

La plantilla se ocupa de cuatro aspectos que también debes considerar en una configuración propia:

1. **`propagation: rslave`** en el montaje bind de medios del contenedor de Paperless; de lo contrario, el contenedor no sobrevive a un reinicio del montaje. Detalles y la trampa de AppArmor que hay detrás: [Montaje de Rclone en el contenedor Docker](/blog/rclone-mount-in-docker-container).
2. **Detener Paperless si falta el montaje.** De lo contrario, escribe documentos en un directorio local vacío, y el montaje que regresa los oculta de forma invisible. La plantilla incluye un script de watchdog.
3. **Una cuenta que pueda iniciar sesión sin supervisión.** En Proton, esto significa guardar la clave TOTP en la configuración de Rclone. Por qué esto no invalida la autenticación de dos factores y cuál es la situación general de Proton en Linux: [Proton Drive en Linux](/blog/proton-drive-linux-status).
4. **Desactivar las tareas programadas de lectura completa** (`PAPERLESS_SANITY_TASK_CRON=disable`), ya que, de lo contrario, la comprobación de integridad lee regularmente todo el conjunto desde la nube.

## Qué deberías valorar antes de usarlo

Un documento recién incorporado permanece durante unos segundos solo en la caché local, hasta que termina la subida. Si la máquina falla justo en esa ventana, falta el archivo. El límite de caché es flexible y puede superarse notablemente durante picos de acceso durante un breve periodo. Además, el backend de Proton de Rclone está oficialmente en beta; con llamadas rápidas a la API mostró síntomas de limitación. Como aún faltan datos a largo plazo de funcionamiento continuo, la plantilla está marcada como experimental.

El artículo sobre metodología explica cómo se obtuvieron los valores medidos, qué fallos se simularon y cómo se puede probar rigurosamente una configuración de este tipo: [Probar montajes en la nube con PDF generados](/blog/cloud-mount-testen-dummy-pdfs).

## Conclusión

Es posible y apto para el día a día usar Paperless-ngx en un disco pequeño con almacenamiento en la nube: apenas dos segundos al abrir por primera vez, después velocidad de caché, la búsqueda y la interfaz permanecen independientes de la nube, y la configuración se recupera sola tras los fallos. No obstante, si solo quieres ahorrar unos pocos gigabytes en un servidor de tamaño normal, deberías hacer cuentas: en mi caso, todo el almacenamiento ocupaba 71 MB y el sistema operativo varios gigabytes. La ventaja no está en el espacio ahorrado de inmediato, sino en que el conjunto puede crecer sin que el disco tenga que crecer con él.

## Fuentes

1.  [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage): la plantilla de este artículo: setup.sh, wizard.sh, archivos Compose, watchdog y guía de adaptación.

2.  [Rclone: Overview of cloud storage systems](https://rclone.org/overview/): los más de 70 servicios compatibles y sus capacidades comparadas.

3.  [Paperless-ngx: Configuration](https://docs.paperless-ngx.com/configuration/): `PAPERLESS_ARCHIVE_FILE_GENERATION`, `PAPERLESS_SANITY_TASK_CRON` y las demás configuraciones utilizadas.

4.  [Paperless-ngx: Administration](https://docs.paperless-ngx.com/administration/): comprobador de integridad, exportación e importación, así como las tareas programadas en segundo plano.
