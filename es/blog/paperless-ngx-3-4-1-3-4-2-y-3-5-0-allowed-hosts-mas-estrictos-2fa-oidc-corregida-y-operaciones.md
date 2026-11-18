---
title: "Paperless-ngx 3.4.1, 3.4.2 y 3.5.0: ALLOWED_HOSTS más estrictos, 2FA OIDC corregida y operaciones masivas asíncronas"
navTitle: "Paperless-ngx 3.5.0"
description: "Paperless-ngx 3.5.0 solo permite el nombre de host de PAPERLESS_URL y localhost cuando se ha definido PAPERLESS_URL, corrige el valor predeterminado de AMR ineficaz de 3.4.x y ejecuta las operaciones masivas de forma asíncrona. Antes de actualizar, compruebe mediante qué nombres de host se accede a Paperless-ngx y añádalos a PAPERLESS_ALLOWED_HOSTS."
date: "2026-11-17"
kategorie: "Paperless-ngx"
timeToRead: "7 min de lectura"
themen:
  - paperless-ngx
produkte:
  - "paperless-ngx"
protokolle:
  - "releases"
slug: "paperless-ngx-3-4-1-3-4-2-y-3-5-0-allowed-hosts-mas-estrictos-2fa-oidc-corregida-y-operaciones"
translationId: article-4c816dd3cd08ff6e
translationOf: paperless-ngx-3-5-0
url: https://rafaelpfister.ch/es/blog/paperless-ngx-3-4-1-3-4-2-y-3-5-0-allowed-hosts-mas-estrictos-2fa-oidc-corregida-y-operaciones
translationSourceHash: 2c1c3db7502c455d8090a922dd17654a159d2d094be4f29d0da51f29eababfb4
translationModel: gpt-5.6-terra
translatedAt: 2026-11-18T03:22:41.597Z
translationReview: automatic
---

# Paperless-ngx 3.4.1, 3.4.2 y 3.5.0: ALLOWED_HOSTS más estrictos, 2FA OIDC corregida y operaciones masivas asíncronas

Desde la versión 3.4.0, el proyecto Paperless-ngx ha publicado tres versiones: las versiones de corrección de errores 3.4.1 el 31 de octubre de 2026 y 3.4.2 el 3 de noviembre de 2026, así como la versión con nuevas funciones 3.5.0 el 17 de noviembre de 2026. Las dos versiones de parche corrigen principalmente errores de visualización de la interfaz web rediseñada y actualizan Django y otras dependencias. La versión 3.5.0 modifica el valor predeterminado de los nombres de host permitidos, corrige un error por el que la configuración introducida en 3.4.0 `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR` no tenía efecto, ejecuta todas las operaciones masivas de forma asíncrona, elimina automáticamente las sesiones caducadas e incorpora flujos de trabajo activables manualmente. Ninguna de las tres versiones menciona un aviso de seguridad con CVE o GHSA. El estado anterior se describe en el artículo [Paperless-ngx 3.4.0: inicio de sesión OIDC sin 2FA duplicada, miniaturas sin ImageMagick](/blog/paperless-ngx-3-4-0). Las actualizaciones están dirigidas a todos los operadores de Paperless-ngx 3.x; se ven especialmente afectadas las instalaciones a las que se accede mediante nombres de host internos y las instalaciones con inicio de sesión OIDC.

## Las versiones de un vistazo

| Versión | Fecha | Tipo | Puntos clave |
|---|---|---|---|
| 3.4.1 | 31.10.2026 | Versión de corrección de errores | Correcciones en el esquema de la API y en la interfaz web, menor uso de memoria en inactividad |
| 3.4.2 | 03.11.2026 | Versión de corrección de errores | Tareas de edición masiva en trabajadores de Celery, instalación del idioma de Tesseract antes de crear el índice, filelock 4.2.6, actualizaciones de Django y dependencias del frontend |
| 3.5.0 | 17.11.2026 | Versión con nuevas funciones | ALLOWED_HOSTS sin comodín cuando se define PAPERLESS_URL, valor predeterminado de AMR corregido, operaciones masivas asíncronas, limpieza automática de sesiones, activador manual de flujos de trabajo, jerarquía de etiquetas sin django-treenode |

## 3.4.1 y 3.4.2: correcciones tras la versión con nuevas funciones

La versión 3.4.1 corrige la ubicación de una corrección del esquema para campos `id` (PR #14604), elimina comprobaciones de permisos duplicadas en el botón de abrir de las tarjetas de documentos (PR #14601), oculta en gran medida los nuevos controles de PDF (PR #14595), desplaza el botón de vista previa en la lista de documentos (PR #14594) y corrige la visualización de la suma de comprobación en modo claro, así como la posición del desplegable de campos personalizados en dispositivos móviles (PR #14590). Además, Paperless-ngx carga más módulos solo cuando son necesarios, lo que reduce el uso de memoria en inactividad (PR #14599).

La versión 3.4.2 incluye once correcciones de errores. Para la operación, son relevantes las siguientes:

- Las tareas de importación desde `bulk_edit` se ejecutan de forma segura en los trabajadores de Celery (PR #14653).
- La creación del índice de búsqueda y del índice LLM espera a que se instalen los paquetes de idioma de Tesseract (PR #14670).
- filelock se actualiza a la versión 4.2.6; las notas de la versión justifican la actualización con correcciones relacionadas con ACL (PR #14656).
- El historial de documentos muestra nombres de etiquetas en lugar de ID (PR #14625), los botones de zoom ya no envían formularios (PR #14624) y el nombre de una vista guardada se conserva al guardar y cerrar (PR #14667).

En cuanto a dependencias, la versión 3.4.2 actualiza Django, django-guardian, django-filter y drf-spectacular-sidecar (PR #14661), turbohtml a 1.15.3 (PR #14662) y pdfjs-dist a 6.4.299 (PR #14644). El proyecto sustituye la biblioteca ngx-bootstrap-icons, que ya no se mantiene, por una solución propia (PR #14665). En los archivos Compose de ejemplo, Gotenberg pasa a 8.38 (PR #14634) y nginx a 1.31.7 (PR #14633). Las notas de las versiones 3.4.1 y 3.4.2 no identifican ninguna de estas entradas como actualización de seguridad.

## 3.5.0: ALLOWED_HOSTS sin comodín

Las notas de la versión 3.5.0 comienzan indicando que algunos usuarios deben adaptar su configuración `PAPERLESS_ALLOWED_HOSTS` (PR #14735). Hasta la versión 3.4.2, el proyecto documentaba para `PAPERLESS_ALLOWED_HOSTS` el valor predeterminado `*`, es decir, todos los hosts. A partir de 3.5.0, según la documentación de configuración:

- Si no se define ni `PAPERLESS_URL` ni `PAPERLESS_ALLOWED_HOSTS`, Paperless-ngx sigue aceptando cualquier host.
- Si solo se define `PAPERLESS_URL`, Paperless-ngx acepta únicamente el nombre de host de esta URL y `localhost`.
- `PAPERLESS_ALLOWED_HOSTS` añade hosts adicionales a `localhost` y al nombre de host de `PAPERLESS_URL`. `localhost` permanece siempre permitido porque Docker lo necesita para la comprobación de estado.
- `PAPERLESS_ALLOWED_HOSTS=*` permite cualquier host, incluso si se define `PAPERLESS_URL`.

Se ven afectadas las instalaciones con `PAPERLESS_URL` definido a las que también se accede mediante otros nombres, por ejemplo, mediante un nombre de host interno, una dirección IP o el nombre del servicio en una red de contenedores a través de la cual otros servicios invocan la API. Django rechazará esas solicitudes tras la actualización. Introduzca todos los nombres adicionales en `PAPERLESS_ALLOWED_HOSTS` antes de actualizar, por ejemplo:

```shell
PAPERLESS_URL=https://paperless.example.com
PAPERLESS_ALLOWED_HOSTS=paperless.intern.example.com
```

Solo debería establecer el valor comodín `*` si Paperless-ngx es accesible exclusivamente en una red aislada; la documentación aconseja expresamente definir los hosts permitidos para instalaciones en Internet abierto.

## 3.5.0: inicio de sesión OIDC y sesiones

Con la versión 3.4.0 llegó la configuración `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, con la que Paperless-ngx omite la consulta local de dos factores cuando el proveedor OIDC informa de un segundo factor de confianza en el claim `amr`. Según el issue #14692, la lista de valores de confianza siempre permanecía vacía en 3.4.x: el código pasaba `mfa` como separador en lugar de como valor predeterminado. Por ello, sin variable definida y también con `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR=mfa`, Paperless-ngx seguía solicitando el segundo factor local. El PR #14693 lo corrige en 3.5.0. El valor predeterminado documentado es `mfa`; los inicios de sesión con contraseña siguen requiriendo el código de Paperless-ngx.

Por tanto, el cambio de comportamiento descrito en el artículo sobre 3.4.0 no se produce realmente hasta 3.5.0: los usuarios con 2FA configurada localmente cuyo proveedor de identidad proporciona `mfa` en el claim `amr` dejan de ver una segunda consulta tras la actualización. Antes de actualizar, compruebe qué valores AMR envía su proveedor. La documentación exige introducir únicamente valores que indiquen que realmente se utilizó un segundo factor al iniciar sesión. Al mismo tiempo, django-allauth se actualiza a 65.21.1 (PR #14672).

Hasta ahora, Paperless-ngx nunca eliminaba las sesiones caducadas de la base de datos. El desencadenante del cambio fue el issue #14721: con `PAPERLESS_AUTO_LOGIN_USERNAME` se creaba una nueva sesión cada 30 segundos que nunca se eliminaba. A partir de 3.5.0, una tarea programada elimina las sesiones caducadas (PR #14722). El momento lo controla `PAPERLESS_SESSION_CLEANUP_TASK_CRON`; el valor predeterminado es `0 3 * * *`, es decir, diariamente a las 03:00.

## 3.5.0: operaciones masivas, flujos de trabajo y etiquetas

- **Operaciones masivas asíncronas:** Todas las operaciones masivas sobre documentos y la edición masiva de metadatos se ejecutan mediante la cola de tareas (PR #14773, #14711) y aparecen en la lista de tareas. Según la documentación de la API, las operaciones masivas en cola responden con `{"result":"OK","task_id":"…"}`; consulte el resultado mediante `/api/tasks/?task_id=<uuid>`. Cuando la tarea masiva ha finalizado, las tareas posteriores, como el reprocesamiento de archivos, solo están programadas, no terminadas. Adapte en consecuencia los scripts que continúen trabajando directamente con el resultado tras la llamada.
- **Activador manual de flujos de trabajo:** Los flujos de trabajo se pueden aplicar manualmente a documentos seleccionados con el nuevo tipo de activador, en la edición masiva y en los detalles del documento (PR #14704).
- **Jerarquía de etiquetas:** Paperless-ngx almacena ahora la jerarquía de etiquetas como lista de adyacencia y prescinde de django-treenode (PR #14612). La actualización incluye una migración de base de datos para ello. El autor midió en un prototipo con 6000 etiquetas en SQLite tiempos significativamente menores, unos 1,3 en lugar de 4409 milisegundos para crear una etiqueta; los valores no han sido verificados de forma independiente. Al eliminar una etiqueta, Paperless-ngx también comprueba los permisos de las etiquetas subordinadas (PR #14687).
- **Otras novedades:** Las notas de los documentos se muestran como Markdown (PR #14626). Un modelo de embedding de Hugging Face almacenado localmente se carga sin conexión al Hub (PR #14728), y el chat de IA también filtra entre más de 32 700 documentos (PR #14754).

## 3.5.0: otras correcciones de errores

- **Argumentos de OCR:** La API rechaza los argumentos que Paperless-ngx pasa por sí mismo a OCRmyPDF (`input_file_or_options`, `output_file` y `sidecar`); se ignoran en `PAPERLESS_OCR_USER_ARGS` (PR #14734). Compruebe si su configuración contiene estos argumentos.
- **Flujos de trabajo:** El filtro de nombres de archivo de los flujos de trabajo programados utiliza una expresión regular compatible con PostgreSQL (PR #14783).
- **Correo:** Los correos electrónicos guardados se escriben con finales de línea CRLF (PR #14745), y la conversión a PDF/A de archivos de correo se realiza una sola vez (PR #14698).
- **Miniaturas:** Paperless-ngx vuelve a generar miniaturas para imágenes sin versión archivada (PR #14741).
- **Permisos y cuentas:** Los permisos de la vista de configuración se han unificado (PR #14709), Paperless-ngx bloquea las filas de base de datos afectadas al registrar el primer usuario (PR #14688) y las contraseñas se muestran con una máscara constante (PR #14784). El autor del PR #14709 declara expresamente que el cambio no constituye una vulnerabilidad.

## Dependencias y seguridad

No existe ningún aviso de seguridad del proyecto sobre 3.4.1, 3.4.2 o 3.5.0 (a fecha de 17 de noviembre de 2026). El aviso más reciente, GHSA-c8f3-pqr2-rr24, del 16 de octubre de 2026, afecta a flujos de trabajo de clasificación por IA en documentos sin propietario y está corregido desde 3.3.2; los detalles figuran en el artículo sobre 3.4.0. Los sistemas con una versión 3.x anterior también cierran esta brecha al actualizar a 3.5.0.

## Problemas conocidos

- **2FA local con OIDC pese a AMR `mfa` (3.4.0 a 3.4.2):** corregido en 3.5.0 (issue #14692, PR #14693).
- **Problemas conocidos de 3.5.0:** Las notas de la versión no mencionan ninguno (a fecha de 17 de noviembre de 2026). La indicación sobre `PAPERLESS_ALLOWED_HOSTS` es un cambio de comportamiento intencionado.

## Procedimiento recomendado

La versión objetivo es 3.5.0. Antes de actualizar, determine mediante qué nombres de host se accede a Paperless-ngx y añádalos a `PAPERLESS_ALLOWED_HOSTS`; compruebe también los valores de `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`. Detenga los procesos en curso, como la ingesta de documentos, y cree una copia de seguridad, tal como exige la documentación de administración. En instalaciones Docker con imagen de Docker Hub, detenga el stack con `docker compose down`, descargue la nueva imagen con `docker compose pull` e inicie con `docker compose up`; las migraciones de base de datos, incluida la nueva migración de la jerarquía de etiquetas, se ejecutarán automáticamente. En instalaciones bare metal, actualice los paquetes de Python con `pip install -r requirements.txt` y ejecute las migraciones en el directorio `src` con `python3 manage.py migrate`; después, `python3 manage.py document_index reindex --if-needed` y `python3 manage.py document_llmindex migrate` actualizan, si es necesario, el índice de búsqueda y el índice LLM. Si actualiza directamente desde 3.3.x, también se aplican las indicaciones del artículo sobre 3.4.0, especialmente las relativas a `pdftoppm`. El artículo [Ejecutar Paperless-ngx con poco espacio de almacenamiento](/blog/paperless-dokumente-clouddienst-auslagern) describe una instalación Docker con almacenamiento de documentos externalizado.

## Fuentes

1.  [GitHub: versión Paperless-ngx v3.5.0](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.5.0): notas de la versión con información sobre `PAPERLESS_ALLOWED_HOSTS`, nuevas funciones y correcciones de errores.

2.  [GitHub: versión Paperless-ngx v3.4.2](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.2): correcciones de errores y dependencias actualizadas.

3.  [GitHub: versión Paperless-ngx v3.4.1](https://github.com/paperless-ngx/paperless-ngx/releases/tag/v3.4.1): correcciones de errores y menor uso de memoria en inactividad.

4.  [API de GitHub: versiones de Paperless-ngx](https://api.github.com/repos/paperless-ngx/paperless-ngx/releases): fechas de publicación de 3.4.1, 3.4.2 y 3.5.0.

5.  [Paperless-ngx: documentación de configuración 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/configuration.md): nuevo comportamiento de `PAPERLESS_ALLOWED_HOSTS`, valor predeterminado de `PAPERLESS_SOCIAL_ACCOUNT_MFA_TRUSTED_AMR`, `PAPERLESS_SESSION_CLEANUP_TASK_CRON` y argumentos de OCR ignorados.

6.  [Paperless-ngx: documentación de configuración 3.4.2](https://github.com/paperless-ngx/paperless-ngx/blob/v3.4.2/docs/configuration.md): valor predeterminado anterior `*` para `PAPERLESS_ALLOWED_HOSTS`.

7.  [GitHub: issue #14692, MFA trusted AMR siempre vacío](https://github.com/paperless-ngx/paperless-ngx/issues/14692): causa y síntomas del valor predeterminado de AMR ineficaz en 3.4.x.

8.  [GitHub: PR #14722, limpieza automática de sesiones](https://github.com/paperless-ngx/paperless-ngx/pull/14722): contexto de las sesiones nunca eliminadas con inicio de sesión automático.

9.  [GitHub: PR #14612, jerarquía de etiquetas como lista de adyacencia](https://github.com/paperless-ngx/paperless-ngx/pull/14612): sustitución de django-treenode, migración y mediciones del autor.

10.  [Paperless-ngx: documentación de la API 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/api.md): operaciones masivas asíncronas y consulta mediante `task_id`.

11.  [Paperless-ngx: documentación de administración 3.5.0](https://github.com/paperless-ngx/paperless-ngx/blob/v3.5.0/docs/administration.md): procedimiento de actualización para Docker y bare metal.

12.  [GitHub: avisos de seguridad de Paperless-ngx](https://github.com/paperless-ngx/paperless-ngx/security/advisories): resumen de los avisos, ninguno sobre 3.4.1 a 3.5.0.
