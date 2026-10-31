---
title: "Paperless-ngx 3.4.0: inicio de sesión OIDC sin 2FA duplicada, miniaturas sin ImageMagick"
navTitle: "Paperless-ngx 3.4.0"
description: "Paperless-ngx 3.4.0 omite la 2FA local cuando el proveedor OIDC informa de un segundo factor de confianza, genera miniaturas de PDF con Poppler en lugar de ImageMagick y actualiza varias dependencias por motivos de seguridad. Antes de actualizar, revise la nueva configuración AMR y, en sistemas Bare Metal, compruebe que pdftoppm esté instalado."
date: "2026-10-30"
kategorie: "Paperless-ngx"
timeToRead: "5 min de lectura"
themen:
  - paperless-ngx
produkte:
  - "paperless-ngx"
protokolle:
  - "releases"
slug: "paperless-ngx-3-4-0-inicio-de-sesion-oidc-sin-2fa-duplicada-miniaturas-sin-imagemagick"
translationId: article-57582656ad9c441b
translationOf: paperless-ngx-3-4-0
url: https://rafaelpfister.ch/es/blog/paperless-ngx-3-4-0-inicio-de-sesion-oidc-sin-2fa-duplicada-miniaturas-sin-imagemagick
translationSourceHash: fe1a5dd3d088e7e544b9448ecc7099233f8287856a5eded7af7488e06c8f290a
translationModel: gpt-5.6-terra
translatedAt: 2026-10-31T03:19:43.649Z
translationReview: automatic
---

# Paperless-ngx 3.4.0: inicio de sesión OIDC sin 2FA duplicada, miniaturas sin ImageMagick

El proyecto Paperless-ngx publicó la versión 3.4.0 el 29 de octubre de 2026, una semana después de la 3.3.4. Esta versión con nuevas funciones incorpora una configuración que elimina la solicitud local de doble factor en los inicios de sesión OIDC cuando el proveedor de identidad ya ha confirmado un segundo factor. Paperless-ngx ahora genera las miniaturas de documentos PDF con las herramientas Poppler; ImageMagick deja de ser una dependencia. También incluye ampliaciones en los flujos de trabajo y los Custom Fields, cambios en las cargas y las notificaciones de la interfaz web, ocho correcciones de errores y actualizaciones de dependencias de frontend y Python que el proyecto clasifica como actualizaciones de seguridad. La versión no menciona un aviso de seguridad propio con CVE o GHSA. Está dirigida a todos los operadores de Paperless-ngx 3.x, especialmente a instalaciones con inicio de sesión OIDC y a instalaciones Bare Metal cuyos paquetes de sistema administra usted mismo.

## Las versiones de un vistazo

| Versión | Fecha | Tipo | Puntos principales |
|---|---|---|---|
| 3.4.0 | 29.10.2026 | Versión con nuevas funciones | omitir la 2FA local con AMR OIDC de confianza, miniaturas PDF mediante Poppler, ImageMagick eliminado, Custom Fields en títulos de flujos de trabajo, actualizaciones de seguridad de dependencias |

## Inicio de sesión OIDC: 2FA local con AMR de confianza

Como novedad, Paperless-ngx evalúa al iniciar sesión mediante un proveedor OIDC el claim `amr` del token de ID (PR #14403). Si contiene uno de los valores de `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, Paperless-ngx omite su propia solicitud de doble factor para ese inicio de sesión. Según la documentación de configuración, el valor predeterminado de la variable es `mfa`. La documentación cita como ejemplos Authelia, que informa de `mfa`, y Pocket ID, que informa de `phr` al usar passkeys. Pocket ID envía el valor `otp` para códigos de un solo uso por correo electrónico; la documentación lo clasifica como no confiable.

```shell
PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR=mfa
```

Puesto que el valor predeterminado ya está configurado, el comportamiento cambia tras la actualización para los usuarios con 2FA local configurada en cuanto su proveedor de identidad entregue `mfa` en el claim `amr`. Por ello, compruebe qué valores AMR envía realmente su proveedor y si estos valores se establecen únicamente ante un segundo factor real. La documentación exige expresamente introducir solo valores que indiquen que realmente se utilizó un segundo factor al iniciar sesión. En paralelo, la versión 3.4.0 actualiza django-allauth a 65.20 y oauthlib a 4.0 (PR #14570).

## Miniaturas PDF con Poppler, ImageMagick eliminado

Hasta ahora, Paperless-ngx llamaba a ImageMagick (`convert -density 300`) para crear la miniatura de un PDF, que renderizaba la primera página mediante Ghostscript a 300 DPI. A partir de 3.4.0, `pdftoppm` de las herramientas Poppler renderiza la primera página, Pillow escala la imagen y la guarda como WebP (PR #14420). Paperless-ngx calcula la resolución a partir del tamaño de página, que determina mediante `pdfinfo`, con un máximo de 300 DPI. Se elimina la alternativa con Ghostscript para las miniaturas, mientras que se mantiene la reparación mediante qpdf. El autor de la PR midió tiempos de ejecución claramente inferiores y menor consumo de memoria; para un documento A0 vacío, aproximadamente 0,21 en lugar de 12,9 segundos. Estos valores no han sido verificados de forma independiente.

Con la PR #14440, ImageMagick desaparece por completo. Un ayudante basado en Pillow se encarga de las tareas restantes, principalmente de eliminar un canal alfa de las imágenes; guarda los archivos WebP sin pérdidas. Por tanto, ya no es necesaria la política de ImageMagick que antes debía ajustarse para el procesamiento de PDF. Ghostscript sigue instalado porque OCRmyPDF continúa necesitándolo.

Para los administradores, esto tiene dos consecuencias:

- **Bare Metal:** `pdftoppm` debe estar disponible. Si falta, Paperless-ngx muestra una advertencia al iniciarse. Compare los paquetes de sistema instalados con la lista de dependencias actualizada en la documentación de instalación.
- **Configuración:** según la PR #14420, `PAPERLESS_CONVERT_MEMORY_LIMIT` y `PAPERLESS_CONVERT_TMPDIR` están obsoletos y no tienen efecto; `PAPERLESS_GS_BINARY` también se ha marcado como obsoleto. Puede eliminar estas entradas de `paperless.conf` o del archivo `.env`.

## Flujos de trabajo, Custom Fields e interfaz web

- **Flujos de trabajo:** las plantillas de título en los flujos de trabajo pueden utilizar Custom Fields (PR #14564). En los campos de etiquetas de los flujos de trabajo, las etiquetas superiores e inferiores pueden seleccionarse de forma independiente (PR #14400).
- **Custom Fields:** las consultas en campos de fecha pueden realizarse de forma relativa (PR #14549).
- **Cargas y notificaciones:** la barra de navegación dispone de un botón de carga (PR #14442). El botón muestra el número de cargas en curso y enlaza a la vista general; disminuye el número de notificaciones (PR #14501, #14500).
- **Interfaz:** se han revisado los detalles de documentos (PR #14485), el distintivo de tareas carga más rápido (PR #14552) y el chat de IA muestra Markdown (PR #14409, #14415).

## Correcciones de errores

- El `document_exporter` se interrumpía al exportar a una carpeta con `--delete` si existía un `manifest.json.tmp` obsoleto (PR #14526). Quienes utilicen el exportador con `--delete` para copias de seguridad deberían comprobar tras la actualización si las exportaciones vuelven a completarse íntegramente.
- Al editar un documento, el índice de búsqueda se actualiza solo después de ejecutar los flujos de trabajo (PR #14558).
- Si un código de barras no genera una separación de documentos, las etiquetas ya no se asignan con retraso (PR #14497).
- Las tareas canceladas o caducadas se muestran de forma más comprensible (PR #14566).
- La vista de esquema de la API ya no resuelve consultas de permisos sobre documentos (PR #14503), y se ha corregido el tipo de lista de los filtros para objetos de documento en el esquema de la API (PR #14524).
- Las denominaciones de versión largas se acortan en el desplegable y se ha unificado el tamaño de los desplegables (PR #14578, #14510).

## Dependencias y seguridad

Las notas de la versión enumeran como actualizaciones de seguridad actualizaciones de dependencias de frontend (PR #14581), así como de brotli, djangorestframework, filelock, msgpack y setuptools (PR #14575). Las notas de la versión no indican qué vulnerabilidades se corrigen en estos paquetes. Los componentes Angular están en la versión 22.2.2. No existe un Security Advisory propio del proyecto para la 3.4.0 (a fecha de 30 de octubre de 2026).

El Advisory más reciente, GHSA-c8f3-pqr2-rr24, del 16 de octubre de 2026, afecta a las versiones 3.0 hasta anteriores a la 3.3.2 y está corregido en la 3.3.2. Con las funciones de IA activadas, los flujos de trabajo de clasificación por IA para documentos sin propietario podían transmitir extractos y metadatos de documentos privados de otros usuarios al proveedor de IA configurado (CVSS 3.1: 4.3, sin CVE). Los sistemas que aún ejecutan una versión 3.x anterior también cierran esta brecha al actualizar a la 3.4.0.

## Procedimiento recomendado

La versión de destino es la 3.4.0. Antes de actualizar, detenga los procesamientos en curso, como la ingesta de documentos, y cree una copia de seguridad, tal como exige la documentación de administración. En instalaciones Docker con imagen de Docker Hub, detenga el stack con `docker compose down`, descargue la nueva imagen con `docker compose pull` e inicie con `docker compose up`; las migraciones de la base de datos se ejecutan automáticamente. Si todo funciona, inicie el stack con `-d` en segundo plano. En instalaciones Bare Metal, instale primero las nuevas dependencias del sistema, especialmente Poppler con `pdftoppm`, actualice los paquetes Python con `pip install -r requirements.txt` y ejecute las migraciones en el directorio `src` con `python3 manage.py migrate`. Antes del primer inicio de sesión OIDC después de la actualización, determine qué valores deben figurar en `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`. El artículo [Ejecutar Paperless-ngx con poco espacio de almacenamiento](/blog/paperless-dokumente-clouddienst-auslagern) describe una instalación Docker con almacenamiento de documentos externalizado. Las notas de la versión 3.4.0 no mencionan Known Issues ni Breaking Changes (a fecha de 30 de octubre de 2026).

## Fuentes

1.  [GitHub: versión v3.4.0 de Paperless-ngx](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.0): notas de la versión con fecha, nuevas funciones, correcciones de errores y dependencias.

2.  [Paperless-ngx: Changelog](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/changelog.md): registro de cambios en la documentación del proyecto con las entradas de 3.3.x y 3.4.0.

3.  [GitHub: PR #14403, omitir la 2FA local para AMR OIDC de confianza](https://github.com/paperless-ngx/paperless-ngx/pull/14403): nueva evaluación del claim AMR en el inicio de sesión OIDC.

4.  [Paperless-ngx: documentación de configuración](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/configuration.md): descripción y valor predeterminado de `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`.

5.  [GitHub: PR #14420, miniaturas PDF mediante Poppler](https://github.com/paperless-ngx/paperless-ngx/pull/14420): cambio a `pdftoppm`, configuraciones obsoletas y valores medidos por el autor.

6.  [GitHub: PR #14440, ImageMagick eliminado como dependencia](https://github.com/paperless-ngx/paperless-ngx/pull/14440): sustitución de las llamadas restantes a ImageMagick por Pillow.

7.  [GitHub: Security Advisory GHSA-c8f3-pqr2-rr24](https://github.com/paperless-ngx/paperless-ngx/security/advisories/GHSA-c8f3-pqr2-rr24): divulgación de información en flujos de trabajo de clasificación por IA, afectadas las versiones 3.0 hasta anteriores a la 3.3.2, CVSS 4.3.

8.  [Paperless-ngx: documentación de administración](https://github.com/paperless-ngx/paperless-ngx/blob/main/docs/administration.md): procedimiento de actualización para Docker y Bare Metal, opción del exportador `--delete`.

9.  [API de GitHub: versiones de Paperless-ngx](https://api.github.com/repos/paperless-ngx/paperless-ngx/releases): fechas de publicación de 3.3.4 y 3.4.0.
