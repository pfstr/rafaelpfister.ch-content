---
title: "Ejecutar Paperless-ngx con poco espacio de almacenamiento: externalizar documentos a un servicio en la nube"
navTitle: "Paperless con servicio en la nube"
description: "Paperless-ngx solo necesita localmente la base de datos, el índice de búsqueda y las miniaturas; los propios documentos pueden residir en un servicio en la nube. Qué reveló la prueba práctica y cómo completar la configuración con la plantilla lista en tres comandos."
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
translationId: article-2f00e7c17fc45664
translationModel: gpt-5.6-terra
translatedAt: 2026-09-03T08:25:05.036Z
translationReview: automatic
translationSourceHash: 1df015c7f06b7e3e850423bc79663fcd1ac13e66ec5ecd46eb430a0dc5ab3ad1
url: https://rafaelpfister.ch/es/blog/paperless-ngx-poco-espacio-documentos-cloud
---

Paperless-ngx guarda sus documentos en un directorio local, y este directorio crece con cada escaneo. Sin embargo, Paperless apenas necesita los archivos en el uso cotidiano: la búsqueda se realiza en la base de datos, la lista muestra miniaturas y el archivo propiamente dicho solo se lee al abrirlo. Por ello, probé si el almacenamiento podía trasladarse a un servicio en la nube. La herramienta para ello es Rclone, con la que los usuarios de Plex llevan años integrando colecciones multimedia completas desde la nube.

El resultado: **funciona en ambos sentidos**, y la configuración se ha reducido entretanto a tres comandos. Este artículo resume los resultados de la prueba y cómo puede configurar usted mismo la instalación. Los detalles técnicos se tratan en artículos independientes enlazados al final: propagación de montajes de Docker, particularidades de AppArmor, autenticación de dos factores y metodología de medición.

## El principio: el almacenamiento activo permanece local, el almacenamiento en frío está en la nube

| Componente | Ubicación | Por qué |
|---|---|---|
| Base de datos (contiene el texto OCR) | local | necesita bloqueo real |
| Índice de búsqueda, miniaturas | local | accesos constantes |
| **Archivos de documentos** | **Nube** | se leen pocas veces |
| Caché (documentos abiertos recientemente) | local, limitada | los accesos repetidos siguen siendo rápidos |

En Paperless, precisamente el nombre del directorio induce a error: `archive/` **no es almacenamiento en frío**, sino que contiene la versión PDF/A que se entrega en cada visualización. A pesar de su nombre, forma parte del almacenamiento activo. Los originales poco necesarios en `originals/` son el verdadero almacenamiento en frío. Si desea ahorrar al máximo, desactive por completo la copia archivada con `PAPERLESS_ARCHIVE_FILE_GENERATION=never`; la búsqueda de texto completo no se verá afectada, ya que el texto se encuentra en la base de datos.

Por cierto, Paperless-ngx no incluye una integración propia con la nube, ni S3 ni `django-storages`. Un montaje de sistema de archivos mediante Rclone es actualmente la única vía, y funciona con cualquiera de los más de 70 servicios compatibles con Rclone. Proton Drive fue mi elección para las pruebas por el cifrado de extremo a extremo; un almacenamiento compatible con S3 es la alternativa más robusta.

## Lo que reveló la prueba

Pruebas realizadas con una instancia aislada de Paperless, 40 PDF de prueba generados (13.9 MB) y una cuenta de Proton dedicada:

| Operación | Resultado |
|---|---|
| Abrir un documento por primera vez (desde la nube) | ~1.8 s |
| Volver a abrir el mismo documento (desde la caché) | ~20 ms |
| Incorporar un documento nuevo, hasta que esté en la nube | ~20 s |
| Lista de documentos, búsqueda de texto completo | 39 ms / 272 ms, funciona también **sin** conexión a la nube |
| Comprobación de integridad (suma de comprobación de cada archivo) | superada, sin discrepancias |
| Fallo del montaje | autorrecuperación sin reiniciar Paperless, verificada |

Así, el espacio de almacenamiento local queda desacoplado del tamaño del archivo documental: la colección puede crecer, el disco no.

## Cómo configurarlo

La configuración completa está disponible como plantilla en GitHub: [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage). En el servidor:

```bash
git clone https://github.com/pfstr/paperless-cloud-storage.git
cd paperless-cloud-storage
sudo ./setup.sh      # einmalig, bereitet den Host vor (einziger Root-Schritt)
./wizard.sh          # geführt: Anbieter wählen, Zugangsdaten, Rundlauf-Test
```

El asistente consulta el servicio en la nube (Proton, S3, Backblaze B2, WebDAV, SFTP o «Not in the list» para cualquier otro servicio de Rclone), comprueba la conexión con una prueba real de subida y descarga, e inicia el contenedor de almacenamiento. Después:

- **Instalación nueva:** `docker compose -f paperless.yml up -d`, listo.
- **Instancia existente de Paperless:** la base de datos y la configuración permanecen intactas; la guía [RETROFIT.md](https://github.com/pfstr/paperless-cloud-storage/blob/main/docs/RETROFIT.md) describe la carga de los documentos existentes y el cambio necesario en su archivo Compose.

He renunciado deliberadamente a una interfaz web. La GUI web de Rclone se utilizó al principio, pero los túneles SSH, CORS y los montajes efímeros la hicieron peor que la línea de comandos a la que debía sustituir. Tres preguntas en la terminal son más rápidas.

## Para que el montaje se mantenga estable en el uso diario

La plantilla se ocupa de cuatro puntos que también debe considerar en una configuración propia:

1. **`propagation: rslave`** en el montaje bind de medios del contenedor de Paperless; de lo contrario, el contenedor no sobrevivirá a un reinicio del montaje. Detalles y el problema de AppArmor subyacente: [Montaje Rclone en el contenedor Docker](/blog/rclone-mount-in-docker-container).
2. **Detener Paperless si falta el montaje.** De lo contrario, escribe documentos en un directorio local vacío, y el montaje que regresa los cubre de forma invisible. La plantilla incluye un script de vigilancia.
3. **Una cuenta que pueda iniciar sesión sin supervisión.** En Proton, esto significa guardar la clave TOTP en la configuración de Rclone. Por qué esto no devalúa la autenticación de dos factores y cuál es la situación general de Proton en Linux: [Proton Drive en Linux](/blog/proton-drive-linux-status).
4. **Desactivar las tareas programadas de lectura completa** (`PAPERLESS_SANITY_TASK_CRON=disable`), ya que de lo contrario la comprobación de integridad lee periódicamente todo el contenido desde la nube.

## Qué debería valorar antes de utilizarlo

Un documento recién incorporado permanece durante unos segundos solo en la caché local, hasta que termina la subida. Si la máquina falla justo durante esta ventana, el archivo se pierde. El límite de caché es flexible y puede superarse temporalmente de forma considerable durante picos de acceso. Además, el backend de Proton de Rclone está oficialmente en beta; con llamadas rápidas a la API mostró síntomas de limitación. Dado que aún faltan datos a largo plazo de la operación continua, la plantilla está marcada como experimental.

El artículo sobre metodología explica cómo se obtuvieron los valores de medición, qué fallos se simularon y cómo se puede probar rigurosamente una configuración de este tipo: [Probar montajes en la nube con PDF generados](/blog/cloud-mount-testen-dummy-pdfs).

## Conclusión

Paperless-ngx en un disco pequeño con almacenamiento en la nube es viable y apto para el uso diario: poco menos de dos segundos al abrir por primera vez, después velocidad de caché; la búsqueda y la interfaz permanecen independientes de la nube, y la configuración se recupera por sí sola tras los fallos. Sin embargo, si solo desea ahorrar unos pocos gigabytes en un servidor de tamaño normal, debería hacer los cálculos: en mi caso, todo el almacenamiento ocupaba 71 MB y el sistema operativo varios gigabytes. La ventaja no está en el espacio ahorrado de inmediato, sino en que la colección puede crecer sin que el disco tenga que crecer con ella.

## Fuentes

1.  [pfstr/paperless-cloud-storage](https://github.com/pfstr/paperless-cloud-storage): la plantilla de este artículo: setup.sh, wizard.sh, archivos Compose, vigilancia y guía de adaptación.

2.  [Rclone: Overview of cloud storage systems](https://rclone.org/overview/): los más de 70 servicios compatibles y una comparación de sus capacidades.

3.  [Paperless-ngx: Configuration](https://docs.paperless-ngx.com/configuration/): `PAPERLESS_ARCHIVE_FILE_GENERATION`, `PAPERLESS_SANITY_TASK_CRON` y el resto de configuraciones utilizadas.

4.  [Paperless-ngx: Administration](https://docs.paperless-ngx.com/administration/): comprobador de integridad, exportación e importación, así como las tareas programadas en segundo plano.
